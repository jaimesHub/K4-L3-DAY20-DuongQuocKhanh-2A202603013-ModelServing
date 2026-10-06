# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-x86_64` · llama.cpp `b10488`
Settings: `threads=2` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5342 | 1129 / 1545 | 126.5 / 153.8 | 9105 / 10536 / 10536 | 7.9 |
| UD-Q2_K_XL | 2.24 | 4124 | 2168 / 2618 | 150.2 / 224.2 | 11865 / 13285 / 13285 | 6.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.18x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**Is UD-Q2_K_XL worth it on this machine?** No — the 0.73 GB size saving (24.6% reduction) does not justify the latency penalties and lack of quality improvement.

### Speed comparison: Q2 is significantly slower

- **TTFT (prefill):** Q2 2168 ms vs Q4 1129 ms → Q4 is **47.9% faster** (or Q2 is 92% slower)
- **TPOT (decode):** Q2 150.2 ms/tok vs Q4 126.5 ms/tok → Q4 is **15.8% faster** (6.7 vs 7.9 tok/s)
- **E2E (end-to-end):** Q2 11865 ms vs Q4 9105 ms → Q4 is **23.3% faster**

The gap widens especially in prefill, which is the user's first impression. TPOT is also slower despite fewer bits, confirming this machine is **dequantization-bound** (not memory-bandwidth-bound): unpacking 2-bit weights on a CPU without AVX-512 costs more than the bytes saved.

### Quality comparison: both models produce correct content, but Q2 is verbose

Tested the same prompt on both servers (port 8080 for Q4, 8090 for Q2):
> "Giải thích chi tiết cách machine learning models làm việc. Trả lời trong 100-150 từ."

| Metric | Q4 (UD-Q4_K_XL) | Q2 (UD-Q2_K_XL) |
|---|---|---|
| **Prefill** | 122 tokens in 13s | 122 tokens in 16s |
| **Output tokens** | 246 tokens | 286 tokens |
| **Output time** | 1 min 9 s | 1 min 22 s |
| **Decode speed** | 3.55 t/s | 3.49 t/s |
| **Structure** | 3 steps: Data, Training, Prediction + mention of Supervised/Unsupervised/Reinforcement Learning paradigms | 5 steps: Data, Preprocess, Training, Evaluation, Deployment + math formula "Hàm ML(X) → Y" |
| **Assessment** | Concise, fluent, meets word limit ~140 words | Longer, exceeds word limit (~170 words), formula feels out of place |

Both are technically correct, but Q4's answer is tighter and more suitable for the requested scope. Q2 does not offer quality advantage to compensate for its speed penalty.

### Measurement variance note

Between separate runs, benchmarks show small variation (first run: Q4 TTFT 1242 ms, decode 7.5 tok/s; second run: 1129 ms, 7.9 tok/s). Small differences (< 5–10%) should be treated as noise; the consistent trend that Q2 is slower than Q4 across both runs is reliable.

### Conclusion

**Use UD-Q4_K_XL on this machine.** It is nearly 2× faster on prefill, maintains consistent decode speed advantage, and produces quality comparable to or better than Q2. The 0.73 GB size saving is irrelevant on a 16 GB system.
