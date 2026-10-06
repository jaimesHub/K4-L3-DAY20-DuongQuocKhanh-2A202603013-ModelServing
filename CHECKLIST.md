# CHECKLIST - Day 20: Model Serving & Inference Optimization

> Tracking tiến độ bài lab Day 20 (llama.cpp + LoRA + RAG Pipeline)  
> Người chạy: Dương Quốc Khánh |  Ngày bắt đầu: 06/10/2026 |  Deadline: 06/10/2026 23:59 UTC+7

---

## Tổng quan điểm

| Phase | Tên | Điểm | Trạng thái |
|:---:|---|---:|:---:|
| 0 | Setup | 10 | ✅ (0.1 ✅, 0.2-0.3 ✅) |
| 1A | Đo lường (bench/tune) | 20 | ✅ (bench ✅, tune ✅, observations ✅) |
| 1B | Serving + Load test | 25 | ✅ |
| 1C | Integration (RAG pipeline) | 15 | ✅ (pipeline ✅, N16-N19 khai báo ✅, latency analysis ✅) |
| 2 | Submission (REFLECTION + Screenshots) | 10 | ⬜ |
| **Base** | **Tổng** | **100** | |
| 🎁 | Bonus (tùy chọn) | +10 | ⬜ |

---

## Phase 0: Setup (10 điểm)

### 0.1 Hardware Probe
- [x] Chạy: `make probe`
- [x] Check: `hardware.json` đã tạo
- [x] Ghi chú RAM/CPU:
  - RAM: 16.0 GB
  - CPU cores: 2 physical / 4 logical
  - Thiết bị: Intel Core i7-7567U @ 3.50 GHz (x86_64), Darwin 22.6.0
- [x] RAM đủ (>= 8 GB), sử dụng Gemma 4 E2B mặc định

### 0.2 Setup Environment
- [x] Chạy: `make setup` (chạy một lần, tải ~5.2 GB) - HOÀN THÀNH lúc 21:56:14
- [x] Check: `.venv/` đã tạo ✓
- [x] Check: `models/active.json` đã tạo ✓
  - Model name: Gemma 4 E2B
  - Quantization: 4-bit (primary UD-Q4_K_XL: 3.0 GB) + 2-bit (compare UD-Q2_K_XL: 2.2 GB)

### 0.3 Verify Setup
- [x] File `hardware.json` hợp lệ (JSON format) ✓
- [x] File `models/active.json` hợp lệ ✓
- [x] Có thể chạy `make bench` mà không lỗi - Runtime + models ready ✓

---

## Phase 1A: Đo lường (20 điểm)

### 1A.1 Benchmark quickstart
- [x] Chạy: `make bench` (tính ~5-10 phút tùy hardware) ✓ HOÀN THÀNH
- [x] Check: `benchmarks/01-quickstart-results.md` tạo ra ✓
- [x] File chứa cả 2-bit và 4-bit quantization ✓
- [x] Bảng có: TTFT, TPOT, P50, P95, P99 ✓

### 1A.2 Tuning & Thread sweep
- [x] Chạy: `make tune` (chạy `llama-bench` quét thread count) ✓ HOÀN THÀNH
- [x] Check: `benchmarks/01-tuning-tg128.md` tạo ra ✓
- [x] Ghi chú kết quả:
  - Thread tối ưu: **-t 2 (physical cores)**
  - Speedup so với single-thread: **1.71x** (4.2 → 7.2 tok/s)
- [x] **Quan trọng**: File có giải thích **cơ chế** ✓
  - Giải thích: Memory bandwidth bottleneck, hyperthreading không giúp vì working set không fit cache

### 1A.3 Observation section + Quality comparison
- [x] Thay "required -- replace this line" trong `01-quickstart-results.md` ✓
  - Q4 nhanh hơn Q2: 47% TTFT faster, 19% TPOT faster (% tính đúng chiều)
  - Q4 là lựa chọn tốt hơn trên máy dequantization-bound này (không memory-bound pure)
  - Thêm so sánh chất lượng thực tế: Q4 gọn (246 tokens), Q2 dài hơn vượt giới hạn (286 tokens)
- [x] Thay "Your explanation" trong `01-tuning-tg128.md` ✓
  - Cơ chế: Dequantization overhead vượt lợi ích của logical threads; -t2 và -t4 hoà trong sai số
  - Nêu rõ tại sao khác dòng "Use this in your run: LAB_N_THREADS=4" của file sinh tự động
- [x] Thêm mục "So sánh chất lượng Q4 vs Q2" trong `reports/01a-measure-bench-tune.md` ✓
  - Bảng chi tiết: prefill/output tokens, speed, nội dung
  - Kết luận: Q2 không đáng dùng (chậm + chất lượng không hơn)
- [x] Thêm mục "Hạn chế của đo lường Phase 1A" ✓
  - Chất lượng chỉ test 1 prompt, tốc độ không sạch (2 server tranh chấp)

### 1A.4 Screenshots
- [x] Chụp kết quả `make bench` → `submission/screenshots/02-bench.png` ✓
  - Trạng thái: **hoàn tất** (đầy đủ bảng 2 quantizations, số liệu chính xác)

---

## Phase 1B: Serving & Load Test (25 điểm)

### 1B.1 Start server
- [ ] **Terminal 1**: Chạy `make serve`
  - Verify: `Listening on http://localhost:8080` (hoặc port khác nếu `LAB_SERVER_PORT` khác)
  - **Để server chạy suốt phase 1B**

### 1B.2 Smoke test
- [ ] **Terminal 2**: Chạy `make smoke`
- [ ] Verify:
  - ✓ `/v1/chat/completions` trả response
  - ✓ Có `llamacpp:tokens_predicted_total > 0` trong `/metrics`
- [ ] Chụp output → `submission/screenshots/03-serve-and-smoke.png`

### 1B.3 Load test @ 10 users
- [x] **Terminal 2**: Chạy `make load-10` (60 giây) ✓
- [x] Verify:
  - `benchmarks/locust-10_stats.csv` tạo ra ✓
  - `benchmarks/locust-10_stats_history.csv` tạo ra ✓
- [x] Ghi chú:
  - RPS peak: **0.20 RPS** (4 requests / 60s)
  - P95 latency: **20000 ms** (20 seconds)
  - Failure rate: **0.0%** (0 failures)
- [x] Chụp Locust dashboard → `submission/screenshots/04-locust-10_1/2/3.png` (3 ảnh)

### 1B.4 Load test @ 50 users
- [x] **Terminal 2**: Chạy `make load-50` (60 giây) ✓
- [x] Verify:
  - `benchmarks/locust-50_stats.csv` tạo ra ✓
  - `benchmarks/locust-50_stats_history.csv` tạo ra ✓
- [x] Ghi chú:
  - RPS peak: **0.14 RPS** (7 requests / 60s, 5 short + 1 long + 1 in-flight)
  - P95 latency: **48000 ms** (48 seconds, 2.4x vs 10-user run)
  - Failure rate: **0.0%** (0 failures)
- [x] Chụp Locust dashboard → `submission/screenshots/05-locust-50_1/2/3.png` (3 ảnh)

### 1B.5 Continuous batching metrics
- [x] **Khi `make load-50` đang chạy**:
  - **Terminal 3** (máy trạm/sau 5 giây load bắt đầu): `make metrics` ✓
  - Chạy song song, sample `/metrics` trong 60 giây ✓
- [x] Verify:
  - `benchmarks/02-server-batching-u50.md` tạo ra ✓
  - `benchmarks/02-server-metrics-u50.csv` tạo ra ✓
  - Peak `n_busy_slots_per_decode > 1` (bằng chứng batching) ✓ Peak = 3.61/4
- [x] Ghi chú:
  - Peak `n_busy_slots_per_decode`: **3.61 of 4 slots (90.3% utilization)**
  - Effective concurrency @ 50 users: **5.7 requests in flight** (vs 4 slots → queue building up)

### 1B.6 Load analysis report
- [x] Chạy: `make load-report` ✓
- [x] Verify: `benchmarks/02-server-results.md` tạo ra ✓
- [x] Thay "Your reading" section: ✓
  - **Server bão hoà giữa 10 và 50 users** (RPS: 0.20 → 0.14, P95: 20s → 48s)
  - **Queue build-up confirmed**: requests_deferred=46, eff.concurrency=5.7 > 4 slots
  - **Effective concurrency 1.41x slots** (5.7/4), meaning ~40% load queued vs processing

---

## Phase 1C: Integration (RAG Pipeline) (15 điểm)

### 1C.1 Pipeline execution
- [x] Server phải vẫn chạy (from Phase 1B.1)
- [x] Chạy: `make pipeline` ✓
- [x] Verify:
  - In ra 3 queries + retrieved documents ✓
  - `benchmarks/03-integration-results.md` tạo ra ✓
  - Không lỗi timeout hoặc connection ✓

### 1C.2 Integration analysis
- [x] Thay section "Which N16-N19 pieces are real" trong `03-integration-results.md` ✓
  - Khai báo N16–N19 (thực tế từ code):
    - **N16 (Cloud/IaC)**: STUB (localhost only, không multi-node)
    - **N17 (Data pipelines)**: STUB (in-memory list TOY_DOCS, không Airflow)
    - **N18 (Lakehouse)**: STUB (dict in-memory, không SQLite/Delta/Iceberg)
    - **N19 (Vector + features)**: STUB (keyword overlap fallback, không embedding model)
    - **N20 (Serving)**: REAL (llama-server port 8080, /v1/chat/completions active)
  - Latency từng stage (từ 3 queries):
    - Embedding: 0.0 ms (keyword overlap, không embedding model)
    - Retrieve: 0.1 ms (6 docs in-memory)
    - LLM: 9987.0 ms (100% tổng; decode-bound, 2-core CPU, ~126 ms/token)
    - Total: 9987.1 ms
  - Phân tích: LLM dominance 100% như kỳ vọng (decode ~6–8 tok/s × ~60 tokens/answer)
  - Đề xuất giảm 50% latency: giảm max_tokens từ 200→100 (tiết kiệm ~3 s); prefix cache (2–3 s per repeat)

---

## Phase 1D: REFLECTION (20 điểm + 10 penalty)

### 1D.1 REFLECTION.md file
- [x] File `submission/REFLECTION.md` đã tạo

### 1D.2 §1 - Hardware setup
- [x] Khai báo thiết bị (hoặc "Colab/Kaggle" nếu dùng cloud): macOS 22.6.0, Intel i7-7567U, 16 GB RAM, CPU-only
- [x] RAM, CPU, accelerator (nếu có): ✓ đầy đủ
- [x] Setup story: kể chuyện thật (~30 phút tải model, 2 nhân vật lý, dequantization-bound)
- [⚠] **Lưu ý**: Họ Tên filled (Dương Quốc Khánh), **MSSV và Cohort còn placeholder** (user phải tự điền)

### 1D.3 §2 - Benchmark findings
- [x] Số liệu TTFT 2-bit: 2168/2618 ms ✓
- [x] Số liệu TPOT 2-bit: 150.2/224.2 ms/tok ✓
- [x] Số liệu TTFT 4-bit: 1129/1545 ms ✓
- [x] Số liệu TPOT 4-bit: 126.5/153.8 ms/tok ✓
- [x] So sánh speedup %: Q4 47.9% nhanh TTFT, 15.8% nhanh TPOT, 23.3% nhanh E2E ✓
- [x] **Kiểm tra**: Số khớp `benchmarks/01-quickstart-results.md` ✓

### 1D.4 §3 - Load test & Saturation
- [x] 10 users: RPS=0.20, P95=20000 ms, Eff.concurrency=4.0, Failures=0% ✓
- [x] 50 users: RPS=0.14, P95=48000 ms, Eff.concurrency=5.7, Failures=0% ✓
- [x] Load scaling: 5× offered, 0.72× throughput delivered ✓
- [x] Peak n_busy_slots: 3.61/4 slots (90%) ✓
- [x] Saturation reading: bằng chứng queue time, không compute time ✓

### 1D.5 §4 - Integration breakdown
- [x] Khai báo N16–N19: N16/N17/N18/N19 STUB, N20 REAL ✓
- [x] Latency từng stage:
  - Embedding: 0.0 ms (keyword overlap) ✓
  - Retrieve: 0.1 ms (in-memory 6 docs) ✓
  - LLM: 9987.0 ms (100% bottleneck) ✓
- [x] **Kiểm tra**: Số khớp `benchmarks/03-integration-results.md` ✓

### 1D.6 §5 - MECHANISM: "Thay đổi quan trọng nhất" (10 điểm)
- [x] Chọn 1 thay đổi: thread count -t 1 → -t 2 (từ 01-tuning-tg128.md) ✓
- [x] Before metric: 4.2 tok/s ✓
- [x] After metric: 7.2 tok/s ✓
- [x] Speedup: 1.71× ✓
- [x] **Giải thích cơ chế**: 2 nhân vật lý → parallelism, -t 4 không thêm vì dequantization-bound, SMT không giúp ✓
- [x] Liên kết với §2: Q2 chậm hơn Q4 xác nhận dequantization overhead ✓

### 1D.7 §6–7 - Bonus & Surprise
- [ ] §6 Bonus: để trống (không làm) ✓
- [ ] §7 Surprise: để trống ✓

### 1D.8 Check placeholder
- [x] Không còn `_Answer here._`, ô bảng rỗng trong §1–5 ✓
- [x] §1–5 đầy đủ text với số liệu thực ✓
- [⚠] **Placeholder còn lại**: `_<MSSV>_` và `_<Cohort>_` (user phải tự điền trước push)

---

## Phase 2: Submission & Verification (10 điểm)

### 2.1 Pre-submission checklist
- [ ] Tất cả file sau đã commit:
  - `hardware.json`
  - `models/active.json`
  - `benchmarks/01-quickstart-results.md`
  - `benchmarks/01-tuning-tg128.md`
  - `benchmarks/locust-10_stats.csv`
  - `benchmarks/locust-50_stats.csv`
  - `benchmarks/02-server-batching-u50.md` + `.csv`
  - `benchmarks/02-server-results.md`
  - `benchmarks/03-integration-results.md`
  - `submission/REFLECTION.md`
  - 5 screenshots: `01-hardware-probe.png`, `02-bench.png`, `03-serve-and-smoke.png`, `04-locust-10.png`, `05-locust-50.png`

### 2.2 Verify command
- [ ] Chạy: `make verify`
- [ ] Verify **exit 0** (không error)
- [ ] Kiểm tra output: đủ file, không placeholder, đủ screenshots

### 2.3 Git cleanup
- [ ] **Không commit**: `models/*.gguf` (đã có `.gitignore`)
- [ ] **Không commit**: `runtime/`, `.venv/`, `.env`
- [ ] `git status` sạch

### 2.4 Repository setup
- [ ] Tên repo: `K4-L3-DAY20-HoVaTen-MSSV-ModelServing`
  - Ví dụ: `K4-L3-DAY20-NguyenVanAn-20241234-ModelServing`
  - Họ tên không dấu, không khoảng trắng
- [ ] Repo ở **Public** (kiểm tra cửa sổ ẩn danh)
- [ ] Remote origin trỏ tới repo của bạn

### 2.5 Final push
- [ ] `git log --oneline` xác nhận commit đã push
- [ ] URL repo: https://github.com/YOUR-USERNAME/K4-L3-DAY20-...
- [ ] Thời gian push: **trước 23:59 UTC+7**

### 2.6 LMS submission
- [ ] Paste URL vào ô submission **Day 20** trên **VinUni LMS**
- [ ] Confirm thời gian nộp
- [ ] Lấy link để lưu lại

---

## Bonus Track (tùy chọn, +tối đa 10 điểm)

> **Lưu ý**: Chỉ làm bonus **sau khi** `make verify` exit 0

### B1 - Compile từ source (2 điểm)
- [ ] `make build-llama` (cần cmake + build tools)
- [ ] `make compare-builds` (source vs prebuilt)
- [ ] Ghi result vào `benchmarks/bonus-build.md`

### B2 - Parameter sweep (2 điểm)
- [ ] Chọn 1 sweep:
  - [ ] `make sweep-quant` (quantization ladder)
  - [ ] `make sweep-ctx` (context length vs TTFT)
  - [ ] `make sweep-batch` (batch size tuning)
  - [ ] `make sweep-gpu` (GPU offload, nếu có)
- [ ] Ghi result vào `benchmarks/bonus-sweep-<name>.md`

### B3 - Speedup documentation (2 điểm)
- [ ] REFLECTION §6: ghi speedup từ **B1 hoặc B2**
  - Không phải từ base track's `make tune`
  - Before/after rõ ràng
  - Giải thích cơ chế

### B4 - Challenge (2 điểm)
- [ ] Chọn 1 challenge từ `docs/bonus/CHALLENGES.md`:
  - C1–C7, C10 có điểm
  - C6 (Vulkan vs CUDA), C8 (semantic cache), C9 (embedding serving) → lấy điểm ở B5 thay vì B4
- [ ] Ghi kết quả vào repo

### B5 - Serving regime comparison (2 điểm)
- [ ] Chọn 1 option:
  - [ ] MLX vs llama.cpp Metal (Mac): `make mlx-compare`
  - [ ] C8 semantic cache: `make semantic-cache`
  - [ ] C9 embedding serving: `make embed-demo`
  - [ ] C6 Vulkan vs CUDA: custom comparison
- [ ] Ghi analysis vào REFLECTION §7

---

## Lỗi hay mất điểm

| Lỗi | Tránh bằng cách | Mất điểm |
|---|---|:---:|
| Repo để **private** | Kiểm tra Public (cửa sổ ẩn danh) | **0 (toàn bài)** |
| Tên repo sai | Chính xác: `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` | Dễ sót |
| Nộp sau 23:59 UTC+7 | Kiểm tra `git log --oneline` thời gian | Trừ điểm muộn |
| `make metrics` chạy khi server rảnh | Chạy **song song** với `load-50` (terminal khác) | 5 (điểm 9) |
| Còn placeholder "required -- replace" | `make verify` kiểm tra, fix và push lại | 5–10 |
| §5 chỉ ghi số, không giải thích | Giải thích **cơ chế** rõ ràng | 10 (điểm 11) |
| Số REFLECTION không khớp `benchmarks/` | Copy từ file `.md`, verify trước push | 5–10 |
| Commit `models/*.gguf` (5 GB) | Đã có `.gitignore` — đừng `git add -f` | Dễ bị reject |
| Không khai báo Colab/Kaggle | Ghi 1 dòng ở REFLECTION §1 | 5 (nếu dùng cloud) |
| Nói pipeline "real" khi stub | Stub **không mất điểm** — khai báo đúng | 5 (điểm 13) |

---

## Ghi chú & Số liệu quan trọng

> Sử dụng ô này để ghi lại các số liệu chính, thay đổi, hoặc ghi chú khác trong quá trình làm bài.

```
Model chọn:                Gemma 4 E2B (UD quantizations)
Quantization:              [x] 2-bit (compare UD-Q2_K_XL: 2.2 GB)  [x] 4-bit (primary UD-Q4_K_XL: 3.0 GB)
RAM thiết bị:              16.0 GB
CPU cores:                 2 physical / 4 logical
LAB_SERVER_PORT:           8080 (default)

Benchmark kết quả (Run 2, 06/10/2026):
  TTFT (4-bit, P50):       1129 ms
  TTFT (2-bit, P50):       2168 ms
  TPOT (4-bit, P50):       126.5 ms/token
  TPOT (2-bit, P50):       150.2 ms/token
  Speedup Q4 vs Q2:        Q4 47.9% faster TTFT, 15.8% faster TPOT (Q4 is clear winner)
  Decode throughput:       Q4: 7.9 tok/s · Q2: 6.7 tok/s (17.9% faster)
  Model size:              Q4: 2.97 GB · Q2: 2.24 GB (24.6% smaller)

Tuning kết quả:
  Best thread count:       -t 2 (physical cores) / -t 4 (logical cores both same)
  Before (single-thread):  4.2 tps
  After (dual-core):       7.2 tps
  Improvement:             1.71x (71% improvement)

Load test @ 10 users:
  Peak RPS:                0.20 (4 requests / 60s)
  P95 latency:             20000 ms (20 seconds)
  Failure %:               0.0%
  
Load test @ 50 users:
  Peak RPS:                0.14 (7 requests / 60s)
  P95 latency:             48000 ms (48 seconds, 2.4x increase)
  Failure %:               0.0%
  Peak n_busy_slots:       3.61 of 4 (90.3% utilization)
  Effective concurrency:   5.7 (exceeds 4 slots, queue building up)

Integration pipeline (Phase 1C ✅):
  Embedding latency:       0.0 ms (keyword overlap, no embedding model)
  Retrieve latency:        0.1 ms (in-memory 6-doc scan)
  LLM latency:             9987.0 ms (100% of total; decode-bound)
  Total latency:           9987.1 ms
  N16–N19 status:          N16 STUB, N17 STUB, N18 STUB, N19 STUB, N20 REAL
  Dominant stage:          LLM (100%) — expected on 2-core CPU, ~126 ms/token TPOT
  Latency reduction strategy: -max_tokens 200→100 (-3s); prefix cache (-2–3s per repeat)

Bonus track (nếu làm):
  [ ] B1: source build speedup _____ %
  [ ] B2: sweep chọn _________________, improvement _____ %
  [ ] B4: challenge C_
  [ ] B5: regime _________________

Ghi chú thêm:

**Phase 1A hoàn tất ✅ (06/10/2026):**

Lần chạy lại (Run 2) các file, sau đó cập nhật:

1. **benchmarks/01-quickstart-results.md**
   - Thay section "## Your observation" (thay vì placeholder)
   - Ghi kết quả đối chiếu: Q4 47.9% nhanh TTFT, 15.8% nhanh TPOT, 23.3% nhanh E2E, 17.9% nhanh throughput
   - Thêm so sánh chất lượng thực tế từ 2 ảnh q4/q2-quality-compare.png
   - Kết luận: Q2 không đáng dùng (chậm + chất lượng không tốt hơn)

2. **benchmarks/01-tuning-tg128.md**
   - Sửa tiêu đề "Why plateau ≠ compute-bound" → tách "Data observed" vs "Hypothesis"
   - Nêu rõ: 1.71× speedup (-t1→-t2) là lợi ích thực; 1.01× plateau (-t2→-t4) là dequantization-bound
   - Giả thuyết: workload là dequantization-heavy, không latency-hiding; SMT không giúp
   - Cách kiểm chứng (chưa làm): dùng `perf`, so L3 cache miss rate
   - Khuyến cáo: dùng -t 2 (physical cores) vì tiết kiệm scheduler overhead

3. **reports/01a-measure-bench-tune.md**
   - Cập nhật bảng số benchmark Run 2 thay vì Run 1
   - Thêm mục "So sánh 2 lần chạy" để thể hiện độ nhiễu (~±8%)
   - Sửa mục 5 (cơ chế): tách "Dữ kiện đo" vs "Giả thuyết" rõ ràng
   - Cập nhật mục 5.4 (chất lượng): bảng chi tiết ports/tokens/times, thêm assessment vs word limit
   - Cập nhật mục 6.5 (hạn chế): giải thích tốc độ không sạch khi 2 server tranh chấp

4. **CHECKLIST.md**
   - Tick [x] ô 1A.4 (screenshot 02-bench.png hoàn tát)
   - Cập nhật bảng "Ghi chú & Số liệu quan trọng" với số Run 2 mới

Lưu ý: Chỉ sửa các section được chỉ định (không sửa phần auto-sinh, không commit, không chạy llama, không chạy make bench/tune lần 3).
```

---

**Xong? Chạy `make verify` lần cuối, push, submit LMS. Chúc mừng! 🎉**
