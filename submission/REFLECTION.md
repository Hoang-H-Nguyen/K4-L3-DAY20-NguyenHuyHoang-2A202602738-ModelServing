# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyen Huy Hoang
**MSSV:** 2A202602738
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux x86_64 (kernel 7.0.0-34-generic)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-12500H
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.3 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU (4 GB), CUDA available
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-vulkan-x64.tar.gz`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (local Linux)
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup chạy local với runtime llama.cpp b10488 và model Qwen3.5 0.8B. Cổng 8080
đã bị chiếm nên server/load/pipeline dùng cổng 8090 qua `LAB_SERVER_PORT=8090`;
không cần cloud fallback. Một số lệnh bind socket cần quyền mạng của môi trường
chạy, nhưng benchmark và smoke test đều hoàn tất.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4110 | 69 / 80 | 7.0 / 7.4 | 507 / 549 / 549 | 143.3 |
| UD-Q2_K_XL | 0.39 | 3064 | 72 / 79 | 7.7 / 8.3 | 559 / 600 / 600 | 129.8 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit nhỏ hơn 0.11 GB (22%) nhưng chậm hơn 9.4%: 129.8 so với 143.3 tok/s. Tôi đã hỏi cùng một câu trên cả hai model; chất lượng gần như tương đương. Vì vậy Q2 phù hợp khi thiếu RAM/dung lượng, còn Q4 đáng dùng hơn nếu ưu tiên tốc độ.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.02 | 2200 | 21000 | 26000 | 8.2 | 0.0% |
| 50 | 3.45 | 13000 | 15000 | 15000 | 41.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** **1.71×**
- **P95 tăng:** **0.71×** (21,000 ms → 15,000 ms)
- **Effective concurrency ở 50 users:** **41.0** so với `--parallel` = **4** slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): **3.97 / 4 slots**

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server đã bão hòa ở tải 50 users: throughput chỉ tăng 1.71× khi tải tăng 5×,
trong khi `requests_processing=4`, `n_busy_slots_per_decode=3.97/4` và
`requests_deferred` đạt 45. Các request dư phải xếp hàng. Để tăng goodput@SLO,
tôi sẽ tối ưu số thread/CPU trước, vì decode đang chiếm đủ 4 slots; tăng thêm
parallel khi bandwidth đã bão hòa có thể chỉ làm queue dài hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stub | No cloud/IaC implementation in this lab repo |
| N17 Data pipeline | stub | No external data pipeline connected |
| N18 Lakehouse | stub | No lakehouse backend connected |
| N19 Vector + features | stub | Keyword-overlap fallback, no vector index |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: **0.0 ms**
- retrieve: **0.0 ms**
- llm: **3621.1 ms**
- **stage chiếm nhiều nhất:** **llm** (**100%** của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Pipeline dùng toy corpus và keyword-overlap fallback nên embed/retrieve đều 0 ms;
LLM chiếm gần như toàn bộ latency (3621.1 ms trên 3621.2 ms), đúng như kỳ vọng
trên CPU. Nếu cần giảm 2×, tôi sẽ tối ưu LLM trước bằng cách giảm output-token
budget và tuning thread/CPU; tối ưu retrieval không tạo khác biệt đáng kể ở
baseline này.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** giảm số thread decode từ 12 xuống 6

```
before:  28.6 tok/s (-t 12)
after:   30.1 tok/s (-t 6)
speedup: 1.05×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Knee nằm ở khoảng 6 threads: throughput đạt 30.1 tok/s, cao hơn mức mặc định
12 threads (28.6 tok/s). Khi tăng lên 16 và 32 threads, throughput giảm còn
22.9 và 17.8 tok/s. Vì decode chủ yếu bị giới hạn bởi memory bandwidth, các
thread bổ sung tranh cùng băng thông và thêm chi phí scheduling/cache contention
thay vì tạo thêm công việc hữu ích. Do đó dùng toàn bộ logical cores làm máy
chậm hơn; mức 6 threads là điểm cân bằng tốt nhất trong sweep này.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** Không làm bonus; chỉ hoàn thành các track bắt buộc §1–§5.

**Numbers:**

```
before:  không áp dụng
after:   không áp dụng
speedup: không áp dụng
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Đã dùng ChatGPT/Codex để đọc hướng dẫn, debug lệnh và hỗ trợ điền báo cáo từ
các số liệu do benchmark, load test và pipeline tạo ra. Không dùng AI để bịa
số liệu hoặc tạo screenshot giả; mọi con số trong báo cáo lấy từ các lần chạy
trên phần cứng được khai báo ở §1.
