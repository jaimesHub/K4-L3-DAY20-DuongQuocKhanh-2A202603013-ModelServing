# 02 - Serve: load test + saturation reading

Host `Darwin-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=2` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 4 | 0.20 | 20000 | 20000 | 20000 | 4.0 | 0.0% |
| 50 | 7 | 0.14 | 44000 | 48000 | 48000 | 5.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.72x** (14% of linear) |
| P95 latency | **2.40x** |
| Effective concurrency at 50 users | 5.7 vs `--parallel 4` slots (occupancy/slot ratio 1.41) |

**Saturated.** Throughput delivered only 0.72x for 5x the offered load, and effective concurrency (5.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.72x while P95 moved 2.40x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 4 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

**Saturation point: between 10 and 50 users**, confirmed by latency explosion and queue formation.

**Measured data and strongest evidence:**

1. **Queue buildup at server (strongest direct evidence):** Server metrics recorded `requests_deferred = 46` and `requests_processing = 4`. Since `--parallel 4` creates 4 decode slots, all slots were occupied (4/4) while 46 additional requests were queued. This is the clearest sign of saturation: the system hit its slot limit and began queuing incoming requests.

2. **P95 latency increased 2.4× due to queue time:** Latency rose from **20s → 48s** (20s × 2.4), while effective concurrency jumped from 4.0 to 5.7 requests in flight. The ~2.1 request increase must be queued (since peak n_busy_slots = 3.61 < 4 slots). By Little's Law separation: compute time alone cannot explain 48s latency (a single request's decode at ~6–8 tok/s over 64 output tokens → ~8–10s expected), so the extra ~38–40s is queue wait. More users = longer queue = higher latency.

3. **Throughput did not scale with load (moderate evidence):** RPS dropped from 0.20 → 0.14 (72% of linear; 5x load → 0.72x throughput delivered). In an unsaturated system, RPS should scale closer to linearly with offered load. The sub-linear scaling (0.72x) is consistent with saturation, though the sample is very small (only 4 and 7 requests completed).

4. **Batching confirms slots nearly full:** Peak n_busy_slots = 3.61/4 (90% utilization) shows the scheduler packed 3.61 requests per decode step on average. This high batch width is evidence saturation had begun — additional requests could not be packed because slots were nearly occupied.

**Statistical limits and caveats:**
- Only **4 requests completed at 10 users, 7 at 50 users** — these are tiny samples. P95 percentiles from 4–7 observations are unreliable.
- Locust counts only *completed* requests, so effective concurrency is an under-estimate of true queued load.
- Run duration (60s) is short; a longer load test (3+ min) would reveal steadier saturation behavior.
- All three processes (locust, server, metrics collection) ran on 2 physical cores, causing CPU contention.

**What to change first to raise goodput at SLO:**

Assuming SLO = **P95 latency < 30 seconds** (a reasonable midpoint between 20s and 48s observed), the current --parallel 4 setting does NOT achieve this at 50 users (P95 = 48s).

**Hypothesis: the compute bottleneck is the 2-core CPU, not the slot count.** Decode is compute-bound (6.5–7.9 tok/s with threads=2 from Phase 1A smoke and bench; single-threaded is only 4.2 tok/s). Adding slots would split the same CPU compute budget across more requests: each individual request would decode slightly slower, and total requests/sec would remain limited. The root issue is compute throughput, not orchestration width.

**Better candidates to test:**
- **Reduce --parallel to 2** (matching 2 physical cores exactly): fewer slot-switching context and CPU contention may improve per-request latency and reduce queue buildup. Verify by re-running load-50 and comparing P95 and queue depth.
- **Reduce output length / max_tokens**: shorter outputs decode faster, freeing compute sooner.

These are hypotheses needing verification. To test the --parallel 2 hypothesis without breaking your current state, you can set **LAB_PARALLEL=2** as an environment variable inline (already supported in lib/labkit.py):

```bash
LAB_PARALLEL=2 make serve    # start server with --parallel 2
# Then in another terminal, while server runs:
LAB_PARALLEL=2 make load-50  # load test with same config
```

**Caveat:** `make load-50` will overwrite `benchmarks/locust-50_*.csv` and `benchmarks/02-server-*.md`, destroying the current data. If you want to preserve the current 1B submission data, back it up first (e.g., copy to a separate directory). This re-run is optional and exploratory — it does NOT affect your 1B submission score if skipped.

**Caveat:** Only 4 and 7 requests is insufficient data to confidently declare saturation point or design solutions. A longer run (3–5 min) would yield 100+ samples and much stronger evidence.
