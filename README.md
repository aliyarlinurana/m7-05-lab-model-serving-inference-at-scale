![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | Model Serving & Inference at Scale

## Overview

You will not stand up a load balancer today. You will plan the capacity. Given a serving scenario, produce the three artifacts an SRE would ask for before letting your service near production: a capacity plan, an SLO/SLI spec, and a load-test plan.

This is a 90-minute design lab. You'll write Markdown tables and YAML. No code.

## Learning Goals

By the end of this lab you should be able to:

- Estimate the replicas and batching needed to meet a throughput and latency target
- Author an SLO/SLI specification a team can monitor against
- Plan a meaningful load test that includes warmup, ramp, and sustained peak

## Setup

Fork and clone the lab repo. You need a Markdown editor and a YAML editor. No runtime.

## The Scenario

You are launching the **Vision Moderation** service from yesterday's lab. Operational facts:

- **Peak traffic:** 300 RPS sustained; 500 RPS for 5-minute spikes
- **Synchronous endpoint** is the dominant path (90% of traffic). p95 latency budget: 250 ms.
- **Batch endpoint** is 10% of traffic, called by partners aggregating uploads.
- **Model**: 180 MB ONNX, ~75 ms median inference on a single CPU core, ~22 ms on a T4 GPU
- **Feature lookup**: a Redis call, p95 ~ 8 ms
- **Pre/post-processing**: ~15 ms combined on CPU
- **Network in+out**: ~10 ms combined
- **Compute budget**: $4,000 / month (cloud bill for serving only; storage and egress paid separately)
- **SLA promise to partners**: 99.5% availability; refunds if p95 latency exceeds 500 ms over a rolling hour

## Tasks

### Task 1 — Capacity plan

Produce `capacity-plan.md` with these sections:

**1. Latency budget breakdown**

A table showing where the 250 ms p95 budget goes, e.g.:

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 5 | |
| Auth + routing | 2 | |
| Payload parse | … | |
| Feature lookup (Redis) | … | |
| Pre-processing | … | |
| Model inference | … | CPU or GPU choice goes here |
| Post-processing | … | |
| Serialization | … | |
| Network out | … | |
| **Headroom** | … | |
| **Total** | 250 | |

Headroom must be positive — show your work.

**2. CPU vs GPU decision**

Pick one. Justify in 4–6 lines with a per-replica throughput estimate and a monthly-cost comparison. Cite reasonable on-demand prices for an instance class (you don't need real-time pricing — within 30% is fine; cite the source).

**3. Replica sizing**

A small table: target RPS, per-replica throughput, replicas needed (with 30% headroom), monthly cost estimate. Include both sustained (300 RPS) and spike (500 RPS) cases. State whether you'll meet the spike via autoscaling, warm overprovisioning, or queue-shedding.

**4. Batching decision**

Will you enable dynamic batching? If yes, what max batch size and max wait window? Justify against the latency budget. Two paragraphs.

### Task 2 — SLO/SLI specification

Write `slos.yaml`:

```yaml
service: vision-moderation
slos:
  - name: availability
    sli: <define exactly what "good" means — which status codes count as good?>
    objective: 0.995          # over a rolling 30-day window
    burn_rate_alerts:
      - severity: page
        burn_rate: 14.4
        window: 1h
      - severity: ticket
        burn_rate: 6
        window: 6h
  - name: latency_p95
    sli: <define the SLI — e.g. proportion of requests with end-to-end latency under 250 ms>
    objective: 0.95
    burn_rate_alerts:
      - …
  - name: latency_p99
    sli: …
    objective: 0.99
    burn_rate_alerts:
      - …
```

Each SLO must include:

- A measurable SLI definition (not just "latency should be low")
- An objective (e.g. 0.995)
- A rolling window
- At least one burn-rate alert with severity, multiplier, and time window

Tie at least one SLO to the **partner SLA** (the 99.5% availability promise with refunds). Show that the internal SLO is stricter than the external SLA.

### Task 3 — Load test plan

Write `load-test-plan.md` (one page). It must include:

1. **Tool choice and why** — k6 / vegeta / Locust / wrk2 — pick one and justify in 2 lines.
2. **Test phases** — warmup duration, ramp profile, sustained peak duration, soak duration. With numbers.
3. **Traffic shape** — synchronous vs batch ratio, payload size distribution, concurrency model.
4. **Pass/fail criteria** — explicit numeric thresholds (p95 ≤ X, error rate ≤ Y, etc.).
5. **Bottleneck checklist** — what you'll inspect on each replica during the test (CPU%, memory, GPU utilization if applicable, downstream Redis latency, etc.).

## Submission

Open a PR to the lab repo with:

```
capacity-plan.md
slos.yaml
load-test-plan.md
```

Paste the PR link as your deliverable.

## Quality bar

You will be reviewed on:

- **Does the latency budget balance?** A column that sums to 251 ms fails.
- **Is the CPU vs GPU decision justified with numbers**, or is it a preference?
- **Are SLO burn-rate alerts properly multi-window** (fast burn for paging, slow burn for tickets)?
- **Does the load test plan have measurable pass/fail criteria**, or is it "we'll see"?
- **Are units consistent?** Don't mix RPS and QPS; don't mix ms and seconds without flagging it.
