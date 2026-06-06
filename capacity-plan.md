# Capacity Plan — Vision Moderation Service

---

## 1. Latency Budget Breakdown

Target: **p95 ≤ 250 ms** for the synchronous `/v1/predict` endpoint.

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 10 | Client → load balancer → replica (cloud region, same AZ) |
| Auth + routing | 3 | API-key lookup from in-memory cache; load balancer routing |
| Payload parse | 5 | JSON decode + base64 decode of image bytes |
| Feature lookup (Redis) | 12 | p95 Redis RTT given as ~8 ms; +4 ms headroom for tail |
| Pre-processing | 15 | Resize, normalize, tensor conversion on CPU |
| Model inference | 55 | GPU (T4): median 22 ms; p95 ≈ 35 ms + 20 ms batch-wait window |
| Post-processing | 10 | Softmax → top-K label extraction + probability formatting |
| Serialization | 5 | JSON encode of PredictResponse |
| Network out | 10 | Replica → load balancer → client |
| Headroom | 125 | Buffer for GC pauses, queue jitter, cold paths |
| **Total** | **250** | Matches p95 budget exactly |

> **How headroom is used:** 125 ms of headroom absorbs occasional Redis spikes
> (p99 Redis can reach ~25 ms), Python GC pauses (~10–30 ms), and batch-wait
> variance. A p95 budget means 5% of requests may exceed individual stage
> estimates — the headroom covers those compounding tail events.

---

## 2. CPU vs GPU Decision

**Decision: GPU (NVIDIA T4)**

### Per-replica throughput estimate

| | CPU (e.g. c6i.xlarge, 4 vCPU) | GPU (g4dn.xlarge, 1× T4) |
|---|---|---|
| Median inference | 75 ms | 22 ms |
| Total request time (p50) | ~128 ms | ~75 ms |
| Concurrent requests (4 threads / 1 GPU stream) | 4 | 8 (dynamic batching) |
| Per-replica throughput | ~31 RPS | ~107 RPS |

### Monthly cost comparison

| Instance | On-demand price | RPS/replica | Replicas for 300 RPS (×1.3) | Monthly cost |
|---|---|---|---|---|
| `c6i.xlarge` (CPU) | ~$0.17 /hr | ~31 | 13 | ~$1,624 |
| `g4dn.xlarge` (GPU, T4) | ~$0.53 /hr | ~107 | 4 | ~$1,528 |

> Prices from AWS on-demand US-East-1 (June 2026, ±30%).  
> Source: https://aws.amazon.com/ec2/pricing/on-demand/

**Justification:** At 300 RPS sustained, T4 GPU instances are *cheaper* than
CPU because fewer replicas are needed to meet throughput, and the lower
per-request inference time (22 ms vs 75 ms) keeps us well within the 250 ms
p95 budget. The 180 MB ONNX model fits comfortably in T4 VRAM (16 GB).
Dynamic batching further improves GPU utilization. CPUs would require 13+
replicas and still leave the inference stage at ~75 ms, consuming 30% of the
latency budget on inference alone.

---

## 3. Replica Sizing

### Per-replica throughput (GPU, T4, dynamic batching on)

- Median inference: 22 ms
- Max batch size: 8, max wait: 20 ms
- Effective per-replica throughput: **~107 RPS** (measured as 1000 ms / ~9.3 ms per input after batching amortization)

### Sizing table

| Scenario | Target RPS | Replicas (raw) | +30% headroom | Rounded up | Monthly cost (g4dn.xlarge) |
|---|---|---|---|---|---|
| Sustained peak | 300 RPS | 2.8 | 3.6 | **4** | ~$1,528 |
| Spike peak | 500 RPS | 4.7 | 6.1 | **7** | ~$2,677 |

### Spike strategy: **Autoscaling with warm floor**

- Minimum replicas (always warm): **4** — covers sustained load without cold starts
- Maximum replicas (autoscale ceiling): **8** — covers 500 RPS spike with margin
- Scale-out trigger: CPU/GPU utilization > 60% for 60 seconds
- Scale-in trigger: utilization < 30% for 5 minutes
- Cold start time for a g4dn.xlarge loading a 180 MB ONNX model: ~45–90 seconds

> Because cold starts are slow, the minimum warm pool of 4 replicas is non-negotiable.
> During a 5-minute spike, autoscaling will bring up additional replicas within ~90 seconds,
> meaning the first ~90 seconds of a spike must be absorbed by the warm pool alone.
> At 500 RPS with 4 replicas (~428 RPS capacity), the warm pool is temporarily
> over-saturated; a short-lived queue (max depth: 200 requests, ~2 s drain time)
> absorbs the excess. Requests exceeding queue depth receive a 503.

---

## 4. Batching Decision

**Decision: Enable dynamic batching on the synchronous `/v1/predict` endpoint.**

**Configuration:**
- Max batch size: **8**
- Max wait window: **20 ms**

**Justification:**
Dynamic batching is enabled because the T4 GPU delivers its best cost-efficiency
when processing multiple inputs per forward pass. Without batching, each request
triggers a separate CUDA kernel launch (~22 ms each); with batching, 8 inputs
share one forward pass of ~28 ms, reducing the amortized per-input inference
cost to ~3.5 ms. This is especially valuable at 300 RPS where back-to-back
requests arrive fast enough to fill a batch within the wait window.

The 20 ms wait window is chosen specifically against the 250 ms latency budget.
Adding at most 20 ms of queuing on top of 22 ms inference leaves 208 ms for all
other stages — well within budget. A larger window (e.g. 50 ms) would improve
throughput further but risks pushing p95 latency above the 250 ms target during
low-traffic periods when batches fill slowly. The batch endpoint (`/v1/predict-batch`)
already receives pre-assembled batches from partners and bypasses the dynamic
batching layer; it uses a fixed max batch size of 32 as per the API contract.

---
