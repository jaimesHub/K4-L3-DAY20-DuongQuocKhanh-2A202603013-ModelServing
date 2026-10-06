# 02 - Continuous batching under load (u50)

Host `Darwin-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.61 of 4 slots (90%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 731 |

Highest sampled value was **3.61 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width (n_busy_slots_per_decode) was **3.61 of 4 slots** (90%), sampled as the average across all decode steps throughout the measurement period. This represents genuine concurrent request packing into shared decode steps, confirming continuous batching was active under load.

Comparing to effective concurrency in `02-server-results.md`: at 50 users, effective concurrency was **5.7 requests in flight**. Since peak n_busy_slots is only 3.61 (less than 4 slots), the excess (5.7 - 3.61 ≈ 2.1 requests) must be queued and waiting, not actively being decoded. The batching metric (3.61) represents actual decode utilization, while effective concurrency (5.7) includes both active + queued requests.

**Which to trust and why:** I trust both — they measure different things. Peak n_busy_slots (3.61) is the grounded server gauge: what the scheduler truly packed per decode step. Effective concurrency (5.7 = RPS × latency) is a derived estimate that includes queue time in latency, so it's naturally inflated over the actual slot utilization. The gap (5.7 vs 3.61) is the queueing evidence: ~2.1 requests were waiting on average, confirming saturation created backlog.
