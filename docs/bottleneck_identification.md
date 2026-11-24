# Bottleneck identification in RoshanFer

Notes on how RoshanFer (and similar systems such as Rajomon) can identify the
bottleneck service, as a first step towards driving an autoscaler.

Requirements:

- The bottleneck must be identified **before drops**.
- The signal should **not rely on CPU utilization** (Breakwater §3.1: CPU covers
  only one kind of contention, e.g. disk-bound requests have high latency with
  low CPU, and CPU cannot tell "100% utilization with no delayed RPCs" from a
  livelock).

Status: design only. No code implemented yet.

## 1. The problem

Inside RoshanFer every endpoint runs at its concurrency limit
`L = N_cores + ceil(N_cores * RTT / s)` (times `N_calls + 1` for non-leaf
endpoints). Under overload all services report `in_flight == limit`, so the
system cannot tell "100% utilized with zero queueing" from "100% utilized with
big queues" by looking at slot occupancy.

Reasons:

- **Closed core.** Past Ingress the system behaves like a closed system
  (Schroeder et al.): queue lengths and latency depend on the population (the
  sum of limits), not on the arrival rate. The size of the overload is only
  visible at Ingress (queue size, `QS-CAP`, drops).
- **Backpressure fills upstream slots.** `in_flight` is decremented on SubReq Out
  only when the sub-request is actually sent after the downstream grant.
  Requests parked in the PPMQueue keep upstream slots occupied. Upstream
  services therefore look exactly as "full" as the real bottleneck.
- So `in_flight == limit` says "congested", not "cause".

Wait-for graph view: draw an edge `i -> d` when requests at `i` are blocked
waiting for credit from `d`. The bottleneck is the **sink** of that graph, the
service that is full but not waiting on anyone. Blocked stations are "busy" in
slot terms but have idle servers (classical queueing networks with blocking).

Dynamic fan-out adds a third case, the **hostage**: a branch that was granted
credit but holds it unused until all branches are granted. Its slots are
occupied by neither its own work nor a downstream.

## 2. The signal: door wait

Breakwater uses queueing delay as its congestion signal. In RoshanFer the
limit keeps the in-app queue at about `N_cores + 1`, so the queue does not
build inside the service. It builds **at the door**: credit requests wait in
the CreditQueue for a grant. That dwell time (the door wait) is RoshanFer's
equivalent of Breakwater's queueing delay.

Properties:

- **Resource-agnostic.** Whatever makes the app slow (CPU, disk, a lock), slots
  are freed more slowly and credit requests wait longer. The signal does not
  need to know which resource it is.
- **Rises before drops.** Drops only happen at Ingress, when its queue exceeds
  the AIMD cap. As load rises the order is:
  1. door wait at the bottleneck,
  2. upstream slots fill with blocked requests,
  3. door wait upstream,
  4. Ingress queue grows,
  5. e2e latency approaches the SLO,
  6. drops.

  Step 1 starts below capacity, because with random arrivals waiting appears
  before rho reaches 1. For a sudden spike (e.g. 1 -> 10 KRPS) the steps happen
  almost together, so the benefit there is localization rather than lead time.
- **P(wait)**, the fraction of credit requests not granted immediately, is a
  CPU-free estimator of slot utilization. It is monotone in load; for a single
  slot with Poisson arrivals it equals rho (PASTA).
- **Already measured.** The server writes the dwell (`now - enter_queue_ts`)
  into Credit Grant bytes 22-25 (`check_credit_transmission`,
  `state.cc:891-911`). The client decodes it in `credit_post_process`
  (`state.cc:492-561`) only to compute `rtt = total - queueing_time`, then
  discards it.

## 3. Attribution: own / blocked / hold

Because of backpressure the door wait spreads upstream, so it says where there
is congestion, not where the cause is. To find the cause, split the time a
request holds a slot at endpoint `i` (Little's law) into:

- **own(i)**: time in the app doing its own work (any resource),
- **blocked(i)**: time waiting for downstream credit (PPMQueue),
- **hold(i)**: credit granted but held unused (dynamic fan-out).

Rule: **the root bottleneck has door wait, and its slot time is dominated by
own time.** A victim has `blocked >= own`. A hostage has large hold time.

Both own and blocked can be computed offline from logs that already exist in
NanoLog builds (`report_latency` in `stats.cc`, timestamps from
`connection.cc`):

- INGRESS `E2E` of `i`: total time of the request at this microservice.
- EGRESS `REQ-FOR` to `d`: from the app emitting the sub-request until it is
  sent, i.e. credit RTT + door wait at `d` (for dfanout, the slowest branch).
- EGRESS `E2E` to `d`: `REQ-FOR` + time spent in `d`.

So:

- `own(i) = E2E_IN(i) - sum_d E2E_EG(i->d)` (max instead of sum for pfanout),
- `blocked(i) = sum_d REQ-FOR_EG(i->d)`.

## 4. Comparison with Rajomon / Dagor

- They have unbounded internal queues, so the queue forms at the bottleneck
  itself and a local delay is enough to localize it. No attribution needed.
- But their delay signal is CPU-specific: the Go runtime
  `/sched/latencies:seconds` metric (goroutine run-queue delay). Confirmed in
  the Dagor code in this repo (`hotel/dagor/queuingDelay.go`); Rajomon is
  assumed to be similar (same group), not verified. A service stuck on disk or
  a lock would not show up.
- RoshanFer is the reverse: the signal is resource-agnostic but needs
  attribution. The attribution is nearly free because the per-edge dwell
  already travels in every Credit Grant.

## 5. Plan

### Sidecar (all under `#ifdef NANO_LOG_ENABLED`)

1. **CWAIT** in `credit_post_process` (`state.cc` ~533, after the rtt
   computation):
   `NANO_LOG(NOTICE, "M# %s CWAIT %s T:T %d", config.name.c_str(), service.c_str(), queueing_time)`.
   Pure door wait per edge, without the RTT that `REQ-FOR` includes. For
   dfanout it shows which branch was slow.
2. **CHOLD** in `fanout_req_management`: when the unused credit return is
   pushed to `credit_return_queue` (`state.cc` ~642), set
   `credit_return->enter_queue_ts = now` (field already exists on `Buffer`). On
   flush (`state.cc` ~655) log `M# <name> CHOLD <ds> T:T <hold_us>`. Gives both
   the number of returned credits and how long they were held.

No change to `exec/nanolog_metrics_plot.py`: both values are in us and are
plotted as latency.

### Classifier (`exec/bottleneck.py`)

Per 200 ms window, per endpoint / microservice:

- `door(i)`: mean `CWAIT` into `i`, plus P(wait),
- `own(i)`, `blocked(i)`: from `E2E` / `REQ-FOR` as above,
- `hold(i)`: from `CHOLD`,
- score = `door(i) / own(i)`: wait in units of service time, dimensionless, so
  comparable across services with us and ms service times. Default
  `theta = 0.5`, to be calibrated on the chain-2 sweep.

| door/own >= theta | blocked vs own | hold  | verdict |
|-------------------|----------------|-------|---------|
| no                | -              | -     | OK |
| yes               | blocked < own  | small | root bottleneck |
| yes               | blocked >= own | -     | victim (follow the edge with the largest CWAIT) |
| yes               | blocked < own  | large | dfanout hostage |

Also report **lead time**: from the first window where the root crosses theta
to the first Ingress `DROP`. This is the "before drops" evidence.

Verdicts at microservice level, door wait at endpoint level. Correct
attribution when a microservice with several endpoints calls the same
downstream is left for later.

### Validation

- chain-2 sweep with `nanolog_debug`: below capacity, the root should have
  `door/own > 0` with no drops; measure lead time. The existing runs in
  `exp_runs_test/` have no `.nanolog.log` files and need a rerun.
- A configuration where the bottleneck moves to another hop should flip the
  verdict.
- `fan-out-dynamic-0-9` should produce the hostage verdict.
- Non-CPU bottleneck (optional, needs a generator change): the test services
  only do `BusyLoop` (`callgraph-framework/utils/busy.go`). Add a mode that
  sleeps inside a global mutex (a serialized resource, like a disk). CPU stays
  low but the verdict should be root.

## 6. Open issue: tight limit vs saturated resource

"door wait + own time" says the endpoint limits throughput. It does not say
whether the resource is saturated or `L` is too tight (e.g. a wrong RTT/s
estimate). For an autoscaler both are fixed by scaling out, but in the second
case resources are wasted.

Resource-agnostic test, BBR-like probing: temporarily raise `L` by 1.

- Throughput rises -> the limit was binding.
- Only own time rises -> the resource is saturated.

RoshanFer controls concurrency directly, so it can do this. Phase 2.
