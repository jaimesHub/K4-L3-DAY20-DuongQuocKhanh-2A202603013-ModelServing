# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Dương Quốc Khánh
**MSSV:** 2A202603013
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS 22.6.0 (Darwin)
- **CPU:** Intel Core i7-7567U @ 3.50 GHz (x86_64)
- **Cores:** 2 physical / 4 logical
- **CPU extensions:** AVX2
- **RAM:** 16.0 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** llama.cpp b10488 (prebuilt binary)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi

**Setup story**: Máy tính cá nhân chạy macOS trên i7-7567U 2 nhân, đủ RAM (16 GB > 8 GB tối thiểu). Model Gemma 4 E2B tải thành công (~30 phút do 2 nhân CPU chậm). Không có GPU, nên đặt `ngl=0` từ đầu. Cài đặt venv và benchmark chạy không lỗi. Thách thức chính: 2 nhân vật lý làm việc dequantization-bound thay vì memory-bandwidth-bound, nên Q2 (2-bit) chậm hơn Q4 (4-bit) mặc dù nhỏ hơn.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5342 | 1129 / 1545 | 126.5 / 153.8 | 9105 / 10536 / 10536 | 7.9 |
| UD-Q2_K_XL | 2.24 | 4124 | 2168 / 2618 | 150.2 / 224.2 | 11865 / 13285 / 13285 | 6.7 |

**Quan sát**: Q4 nhanh hơn Q2 về TTFT (47.9% nhanh hơn: 1129 vs 2168 ms), TPOT (15.8% nhanh hơn: 126.5 vs 150.2 ms/tok), và E2E (23.3% nhanh hơn). **Không đáng dùng Q2** trên máy này. Dù Q2 nhỏ hơn 0.73 GB (24.6%), nhưng do máy là dequantization-bound (2 nhân, no GPU), chi phí unpack 2-bit vượt lợi ích kích thước. Kiểm tra chất lượng: Q4 gọn gàng (246 tokens, trúng yêu cầu ~140 từ), Q2 dài hơn (286 tokens, vượt limit ~170 từ) với format lạ. Cả hai đúng nhưng Q4 tốt hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.20 | 20000 | 20000 | 20000 | 4.0 | 0.0% |
| 50 | 0.14 | 44000 | 48000 | 48000 | 5.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.72×
- **P95 tăng:** 2.40×
- **Effective concurrency ở 50 users:** 5.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.61 / 4 slots

**Saturation reading**: **Server bão hoà giữa 10 và 50 users.** Bằng chứng: (1) throughput chỉ đạt 0.72× dù load tăng 5× (sub-linear = saturation); (2) P95 tăng 2.4× thay vì theo RPS, chứng tỏ lệ queue time tăng nhanh; (3) effective concurrency (5.7) vượt 4 slots → ~2 request đang queue. Peak n_busy_slots=3.61/4 (90%) xác nhận slot gần full. 

**Phân tách latency (ước lượng có điều kiện):** Compute-một-mình suy từ Phase 1A (~8–10 s/request ở 64 token output). Median latency 20 s ở 10 users và 44 s ở 50 users → phần chênh (~24 s thêm ở 50 users) được cho là queue/contention, nhưng chưa đo compute dưới tải. Hạn chế: chỉ 4 requests hoàn thành ở 10 users, 7 ở 50 users (mẫu quá nhỏ); locust tính chỉ completed requests; server và client chạy chung 2 nhân vật lý. Server metrics `requests_deferred=46` khi `requests_processing=4` xác nhận queue tồn tại.

Để nâng goodput@SLO, sẽ **giảm output length** (rút ngắn decode time, giải phóng slot nhanh hơn) hoặc **giảm --parallel** rồi mới xét tăng đôi theo nhân vật lý.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | STUB |
| N17 Data pipeline | STUB |
| N18 Lakehouse | STUB |
| N19 Vector + features | STUB |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 9987.0 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection**: Bottleneck hoàn toàn là LLM decode (9987 ms), như kỳ vọng. Embedding 0.0 ms vì keyword overlap (không model), retrieve 0.1 ms vì 6 docs in-memory. Mỗi câu trả lời ~60 token × 126 ms/token (TPOT từ §2) ≈ 7.5 s, vậy 9.9 s khớp. 

Bước LLM gọi llama-server trên cổng 8080 là **thực tế (REAL)**, không stub; phần này hoạt động với `/v1/chat/completions` endpoint. 

Để giảm latency LLM: latency = prefill + (output_tokens × TPOT~126.5 ms). (1) **Giảm số token sinh ra**: prompt yêu cầu trả lời ngắn hơn hoặc cắt context. Ví dụ: mỗi 10 token bớt đi ≈ 1.27 s (tính từ TPOT p50 126.5 ms, chỉ là ước lượng). Các câu trả lời hiện tại 30–80 token (trần max_tokens=200 không bị chạm). (2) Prefix cache nếu prompt repeat (2–3 s/lần). Không nên dùng Q2 vì chậm hơn trên máy dequantization-bound.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng thread count từ `-t 1` (single-threaded) lên `-t 2` (physical cores)

```
before:  4.2 tok/s
after:   7.2 tok/s
speedup: 1.71×
```

**Điều số liệu cho thấy:**
- `-t 1` → `-t 2`: 4.2 → 7.2 tok/s = **1.71× speedup** (71% improvement)
- `-t 2` → `-t 4`: 7.2 → 7.2 tok/s = **1.01× (plateau, không tăng thêm)**
- Q4 nhanh hơn Q2: 7.9 vs 6.7 tok/s (Q2 chậm hơn 15.8% dù kích thước nhỏ hơn)

**Giả thuyết giải thích:**
Máy tính có 2 nhân vật lý (i7-7567U). Chạy `-t 1` chỉ dùng 1 nhân = một nửa công suất. `-t 2` kích hoạt nhân thứ hai → parallelism thực, tăng 1.71× (không phải đủ 2× do chi phí).

Tại sao 1.71× thay vì 2×? Ít nhất 2 khả năng chưa phân biệt được:
1. **Giới hạn tài nguyên chung (cache/bandwidth):** Dù 2 nhân độc lập, L3 cache (dung lượng hạn chế) phục vụ cả hai. Khi cả 2 nhân decode song song, prefetch bị tranh chấp → stall tăng → tổng throughput < 2×.
2. **Tác động thermal/xung nhịp:** Laptop i7-7567U tiêu thụ 15 W cơ bản. Cả 2 nhân chạy cùng lúc → tỏa nhiệt tăng → hạ xung/ throttle động để tránh quá nhiệt → clock giảm dưới max 3.5 GHz → latency tăng.
3. **Overhead đồng bộ luồng:** KV cache và weights được chia sẻ; cần synchronize giữa 2 luồng → overhead → không hoàn toàn song song.

Không khẳng định con số (cache size, thermal limit) hay "context-switch" nếu chưa đo được; những con số trên là ước lượng dựa trên kiến thức chung, không từ bộ đo.

Thêm `-t 4` (hyperthreading) không giúp (vẫn 7.2 tok/s) vì **workload này dequantization-bound**: mỗi luồng phải unpack weights tại runtime; chi phí bit-manipulation (chuyển đổi 2-bit/4-bit → FP32) vượt lợi ích của SMT (latency-hiding). Số liệu từ §2 xác nhận: Q2 (2-bit) chậm hơn Q4 (4-bit) 15.8% dù kích thước nhỏ hơn → dequantization overhead đáng kể, không memory-bandwidth-bound.

**Cách kiểm chứng** (không làm):
- Đo `perf` để so sánh instruction retirement / memory stall cycles giữa `-t 2` vs `-t 4`
- Quan sát xung nhịp (clock frequency) khi 1 vs 2 nhân chạy
- So tg128 (decode throughput) giữa các quantization khác (Q3, Q5) với thread sweep

**Kết luận**: `-t 2` là tối ưu trên máy này; `-t 4` không thêm lợi ích và nên tránh (context-switch overhead không đáng).



---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0** (chuẩn bị chạy)
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude (Anthropic): Hỗ trợ điền REFLECTION.md §1–5, trích dẫn số liệu từ benchmarks/*.md, giải thích cơ chế trong §5. Không viết code lab, không chạy lệnh make.
