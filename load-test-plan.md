# Load Test Plan — Vision Moderation Service

---

## Tool Choice

**k6** (Grafana k6 v0.50+)

k6 is chosen because it is scriptable in JavaScript, natively outputs p50/p95/p99
histograms and error rates without post-processing, supports ramping virtual-user
profiles out of the box, and integrates directly with Prometheus/Grafana for
real-time dashboards during the test. vegeta is faster for raw HTTP throughput
but offers less flexible scripting for mixed traffic shapes (sync + batch ratio).

---

## Test Phases

| Phase | Duration | RPS Target | Purpose |
|---|---|---|---|
| Warmup | 2 min | 50 RPS | Allow replicas to finish JIT compilation and GPU warm-up; results discarded |
| Ramp-up | 5 min | 50 → 300 RPS | Find the knee of the curve; identify where latency first degrades |
| Sustained peak | 10 min | 300 RPS | Validate steady-state SLOs under production load |
| Spike | 5 min | 500 RPS | Validate autoscaling response and queue behaviour |
| Ramp-down | 2 min | 500 → 50 RPS | Confirm graceful scale-in; no error spike during teardown |
| Soak | 30 min | 300 RPS | Detect memory leaks, connection pool exhaustion, Redis TTL issues |

**Total test duration: ~54 minutes.** Results from the warmup phase are excluded
from all pass/fail calculations.

---

## Traffic Shape

### Endpoint ratio

| Endpoint | Share | Rationale |
|---|---|---|
| `POST /v1/predict` (sync) | 90% | Dominant path per scenario spec |
| `POST /v1/predict-batch` (sync batch) | 10% | Partner bulk uploads |

### Payload size distribution (sync endpoint)

| Image size (decoded) | Share | Notes |
|---|---|---|
| ≤ 100 KB | 40% | Thumbnails, avatars |
| 100 KB – 1 MB | 45% | Typical user uploads |
| 1 MB – 5 MB | 15% | High-resolution originals |

Payloads are pre-generated and stored as base64 strings; k6 selects them
randomly per-request using a weighted distribution array to avoid filesystem
I/O skewing results.

### Batch payload distribution

Each batch request contains 8–16 items drawn from the same image size
distribution above. Batch size is sampled uniformly from [8, 16].

### Concurrency model

- Virtual users (VUs): scaled proportionally to RPS × avg_request_duration
  - At 300 RPS with p50 ~75 ms: ~23 concurrent VUs
  - At 500 RPS spike: ~38 concurrent VUs
- Connection model: persistent HTTP/1.1 keep-alive; each VU holds one connection
- `top_k` parameter: fixed at 5 for all requests (matches default)

---

## Pass / Fail Criteria

All thresholds are evaluated over the **sustained peak phase** (10 min at 300 RPS).
The spike phase has separate thresholds noted below.

| Metric | Threshold | Phase | Action on failure |
|---|---|---|---|
| p95 latency (`/v1/predict`) | ≤ 250 ms | Sustained | **FAIL** — abort and investigate |
| p99 latency (`/v1/predict`) | ≤ 500 ms | Sustained | **FAIL** — abort and investigate |
| p95 latency (`/v1/predict`) | ≤ 400 ms | Spike | **WARN** — log, do not abort |
| HTTP 5xx error rate | ≤ 0.5% | Sustained | **FAIL** |
| HTTP 5xx error rate | ≤ 2.0% | Spike (first 90 s) | **WARN** (autoscaling window) |
| HTTP 5xx error rate | ≤ 0.5% | Spike (after 90 s) | **FAIL** |
| Soak p95 latency drift | ≤ 10% vs sustained baseline | Soak | **FAIL** — memory leak suspect |
| Soak error rate | ≤ 0.5% | Soak | **FAIL** |
| Autoscale replica count | ≥ 7 replicas within 120 s of spike start | Spike | **FAIL** |

---

## Bottleneck Checklist

Collect the following metrics from **every replica** during the test using
Prometheus + node_exporter / DCGM exporter. Sample every 10 seconds.

### Per-replica (GPU instances)

- [ ] **GPU utilization %** — target 60–80% at sustained load; > 95% indicates saturation
- [ ] **GPU memory used (MB)** — should stay below 4 GB; spike above signals batching misconfiguration
- [ ] **CPU utilization %** — pre/post-processing is CPU-bound; > 85% means CPU is the bottleneck, not GPU
- [ ] **Memory (RAM) used %** — watch for growth over the soak phase (memory leak signal)
- [ ] **Active HTTP connections** — should match expected VU count; spikes indicate connection leak
- [ ] **Request queue depth** — if > 50 for > 10 s, replicas are saturated

### Downstream: Redis

- [ ] **Redis command latency p95** — alert if > 15 ms (our budget allows 12 ms p95)
- [ ] **Redis connection pool saturation** — pool exhaustion causes latency spikes
- [ ] **Redis eviction rate** — non-zero evictions mean the cache is too small

### Load balancer / network

- [ ] **Load balancer p95 latency** — end-to-end as seen by the balancer (includes replica processing)
- [ ] **Bytes in/out per second** — validate against network throughput limits of instance type
- [ ] **4xx / 5xx rate by endpoint** — separate tracking for sync vs batch vs async

### Autoscaler

- [ ] **Time from spike start to Nth replica Ready** — must be ≤ 120 s for PASS
- [ ] **Error rate during scale-out window** — expected ≤ 2% in first 90 s
- [ ] **Scale-in behaviour** — confirm no premature scale-in during 5-min spike

---

## Deliverable

After the test, produce a one-page report containing:

1. A p95 latency vs RPS curve (from ramp-up phase)
2. Error rate timeline (all phases)
3. GPU utilization heatmap (all replicas, sustained phase)
4. Pass/fail verdict against every threshold in the table above
5. Recommendation: current replica count and batch config are sufficient / need adjustment

---
