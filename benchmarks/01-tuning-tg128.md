# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-x86_64` · llama.cpp `b10488`
CPU: **2 physical · 4 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 4.2 | 59% |
| 2 | 7.2 | 99% |
| 4 | 7.2 | 100% |

**Best**: `-t 4` at 7.2 tok/s
**Slowest tested**: `-t 1` at 4.2 tok/s (1.71x spread)
**Against the physical-core default** (`-t 2`, 7.2 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

**Knee location: -t 2 (physical cores); plateau at -t 4 (inside measurement error)**

### Data observed

| Finding | Measurement |
|---|---|
| **-t 1 → -t 2 speedup** | 4.2 → 7.2 tok/s = **1.71× (71% improvement)** |
| **-t 2 → -t 4 speedup** | 7.2 → 7.2 tok/s = **1.01× (plateau, within noise)** |
| **Supporting evidence** | Q2 slower than Q4 despite fewer bits (Q4: 7.9, Q2: 6.7 tok/s) |

### Why the knee exists at -t 2 (physical cores)

1. **Single-threaded (-t 1) → Dual-core (-t 2): 1.71× real speedup**
   - Adding the second physical core brings true parallelism
   - Doubling active cores roughly doubles throughput ✓
   - Single core was underutilized

2. **Dual-core (-t 2) → Quad-thread (-t 4): 1.01× plateau (no gain)**
   - Both benchmarks reach **7.2 tok/s** — effectively tied
   - Logical threads on the same physical core share ALU, cache, and memory bus
   - This machine's bottleneck is not latency-hiding (where SMT helps) — it is **compute and dequantization cost**

### Hypothesis: This machine is dequantization-bound, not pure memory-bandwidth-bound

**Evidence:**
- Q2 (UD-Q2_K_XL, fewer bits) is **18.8% slower** in decode than Q4, not faster
- If memory bandwidth were the only bottleneck, smaller model should be advantaged
- Instead, the extra dequantization work per token (unpacking 2-bit vs 4-bit) on a CPU without AVX-512 costs more than it saves

**Mechanism (hypothesis, not directly measured):**
- Each thread must unpack weights at runtime: Q2 requires more bit-manipulation instructions per matmul
- L3 cache working set (model + KV cache) exceeds shared capacity; prefetch cannot hide the stall
- Context-switch overhead between logical threads on the same core ≥ any latency-hiding benefit
- **Implication:** Adding more threads does not hide memory stalls; it only adds contention

### How to verify (not done here)
- Measure actual instruction retirement / memory stall cycles with `perf`
- Compare L3 cache misses between `-t 2` and `-t 4`
- Profile dequantization cost per token in isolation
- Test different quantizations with thread sweep (Q3, Q5, Q6) to confirm Q2 penalty is not an outlier

### Recommendation

**Use `-t 2` (physical core count)** in production.

- `-t 2` and `-t 4` are indistinguishable within measurement variance (both 7.2 tok/s)
- **Physical cores preferred** because:
  - Fewer context switches → lower overhead
  - Better cache locality (each core keeps its working set)
  - Simpler scheduler behavior
  - No SMT benefit on this machine anyway

**Why this differs from the auto-generated line "Use this in your run: LAB_N_THREADS=4":**
- The script suggests `-t 4` because it appears in the "Best" numerically
- But `-t 2` is tied; human reasoning prefers physical cores for this dequantization-bound workload
