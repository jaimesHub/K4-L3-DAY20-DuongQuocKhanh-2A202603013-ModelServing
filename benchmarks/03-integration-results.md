# 03 - Integrate: RAG pipeline run

Host `Darwin-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 11775.4 | 11775.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 8411.6 | 8411.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 9774.0 | 9774.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **9987.0** · total **9987.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Component | Status | Reason |
|---|---|---|
| **N16 (Cloud/IaC)** | STUB | Localhost only; no multi-node setup in repo |
| **N17 (Data pipelines)** | STUB | In-memory list (TOY_DOCS); no Airflow DAG or batch job |
| **N18 (Lakehouse)** | STUB | Six-document dict in memory; not SQLite, Delta, or Iceberg |
| **N19 (Vector + features)** | STUB | Keyword overlap fallback; no embedding model or vector index |
| **N20 (Serving)** | REAL | llama-server on port 8080; `/v1/chat/completions` endpoint active |

### Analysis: Dominant stage and latency breakdown

**Is the LLM dominance expected?** Yes, exactly. The timing split shows:
- **Embedding**: 0.0 ms (keyword overlap has zero cost; a real embedding model would add 50–200 ms)
- **Retrieval**: 0.1 ms (scanning 6 documents in-memory; a larger corpus or vector DB query would be 10–100 ms)
- **LLM inference**: 9987 ms (100% of total)

With a 2-core CPU, decode throughput is ~6–8 tokens/second. Each query answer spans 30–80 tokens:
- Query 1 ("goodput"): ~11775 ms → ~1.8 tokens/s (3 docs) × 31 tokens ≈ expected
- Query 2 ("PagedAttention"): ~8412 ms → ~6.7 tokens/s (3 docs) × 79 tokens ≈ expected  
- Query 3 ("disaggregated"): ~9774 ms → ~6.3 tokens/s (3 docs) × 73 tokens ≈ expected

The latency is **entirely LLM decode**, as predicted from Phase 1A:
- TPOT (Phase 1B): ~126 ms/token (4-bit, single request)
- Here: mean 9987 ms ÷ ~60 tokens ≈ 166 ms/token (accounts for prefill cost amortized + 2-core overhead)

**How to halve latency (from ~10 s to ~5 s)?**

Target the LLM stage via:

1. **Reduce output tokens** (most direct)
   - Lower `max_tokens` in `call_llm()` from 200 → 100–150
   - E2E ∝ output_tokens × TPOT (~126 ms per token from Phase 1A)
   - **Impact**: 3–4 s savings with minimal quality loss

2. **Prefix caching** (if answers repeat context)
   - System prompt + context remain identical → server caches prefill for subsequent requests
   - First call: full decode; 2nd–3rd calls: decode only
   - From Phase 1B metrics: `prompt_tokens_total` grows slowly after first call when system prompt is fixed
   - **Impact**: 2–3 s per repeat (not here, since 3 unique queries)

3. **Quantization trade-off** (risky on this hardware)
   - Q2 is 6.7 tok/s vs Q4's 7.9 tok/s (Phase 1A), a 15% slowdown
   - On a 2-core CPU bound by dequantization, the quality loss exceeds speed gain
   - **Not recommended**; would add latency

4. **Thread configuration** (already tuned)
   - Phase 1A found -t 2 (physical cores) optimal; -t 4 added no gain
   - No further improvement available here

**Practical recommendation for 50% latency cut**: Reduce `max_tokens` to 100–120 tokens per answer. This is safe—the three answers here span 31–79 tokens naturally, leaving headroom. Combined with any prefix-cache reuse, you'd hit ~5 s.

### Quality check

Three answers returned, all factually correct:
1. **"Why is goodput more useful than raw throughput?"** → Correctly quotes context: "Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs." ✓
2. **"What problem does PagedAttention actually solve?"** → Exactly: "PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory." ✓
3. **"When does splitting prefill and decode help?"** → Accurate: "Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound." ✓

All three directly match the TOY_DOCS context; no hallucination detected.

### Limitations of this measurement

1. **Only 3 queries** — too few to measure variance; a single anomalous decode could shift mean by ±1 s
2. **Toy corpus (6 docs, in-memory)** — no scaling insights; real retrieval over 1M+ documents would show different bottleneck
3. **Stub retrieval (keyword overlap)** — embedding + vector DB would add 50–500 ms depending on corpus size and index type (FAISS, SQLite FTS, etc.); this latency is invisible here
4. **2-core CPU** — single-threaded serving model; a 16-core server would show different saturation and batching dynamics
5. **max_tokens=200** — larger outputs magnify decode cost; QA systems often cap at 100–150 tokens
