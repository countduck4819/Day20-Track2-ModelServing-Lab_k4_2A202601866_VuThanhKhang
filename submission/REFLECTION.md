# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** **Vũ Thành Khang**
**Cohort:** **K4**
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** 11th Gen Intel Core i5-1135G7 @ 2.40 GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** không được `detect-hardware` ghi nhận trong `hardware.json`
- **RAM:** 15.7 GB
- **Accelerator:** Vulkan (device present)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** `UD-Q4_K_XL` (primary) + `UD-Q2_K_XL` (compare)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup chạy local với backend Vulkan và đủ RAM cho Gemma 4 E2B. Tôi phải sửa ký tự
Unicode trong `lab.ps1` để Windows PowerShell 5.1 parse đúng. Cổng 8080 đã bị IIS
chiếm và trả HTTP 500, nên tôi chuyển `llama-server` và các client sang cổng 8090.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 46690 | 901 / 6975 | 132.7 / 155.4 | 8820 / 15406 / 15406 | 7.5 |
| UD-Q2_K_XL | 2.24 | 20275 | 1240 / 12049 | 171.2 / 203.3 | 11433 / 24859 / 24859 | 5.8 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 0.73 GB (24.6%) nhưng decode chậm hơn 22.7%; Q4 nhanh hơn 1.29× và có
TTFT/E2E tốt hơn. Vì vậy Q4 đáng dùng hơn trên máy này, trừ khi bắt buộc tiết kiệm
RAM/đĩa. Tôi chưa chạy cùng câu hỏi trên hai quant nên không khẳng định khác biệt
chất lượng.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.20 | 30000 | 54000 | 54000 | 6.2 | 0.0% |
| 50 | 0.23 | 36000 | 56000 | 56000 | 7.8 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.16×
- **P95 tăng:** 1.04×
- **Effective concurrency ở 50 users:** 7.8 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.88 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa ở hoặc dưới 50 users: tải tăng 5× nhưng RPS chỉ tăng 1.16×, batch
width đạt 3.88/4 và 46 request bị deferred. Effective concurrency 7.8 gồm cả hàng
đợi. P95 mẫu nhỏ chưa phản ánh hết request còn chờ. Tôi sẽ thử tăng `--parallel`
từ 4 lên 8 rồi đo lại goodput@SLO, vì bốn slot hiện đều bận; chỉ giữ thay đổi nếu
latency SLO tốt hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost | stub |
| N17 Data pipeline | in-memory list | stub |
| N18 Lakehouse | `TOY_DOCS` dictionary | stub |
| N19 Vector + features | keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 7394.4 ms
- **stage chiếm nhiều nhất:** LLM (xấp xỉ 100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck đúng như kỳ vọng, nhưng tỷ lệ gần 100% còn cao hơn dự đoán;
retrieval chỉ mất 0.2 ms. Muốn giảm latency 2×, tôi sẽ tối ưu serving/model và giảm
generation budget ở LLM. Tối ưu retrieval dưới một mili-giây gần như không thay đổi
tổng latency.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** giảm số CPU threads từ `-t 16` xuống `-t 2`

```
before:  7.6 tok/s (-t 16)
after:   9.0 tok/s (-t 2)
speedup: 1.18×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Kết quả tốt nhất xuất hiện ở 2 threads, không phải tại 4 physical cores. Decode của
mô hình quantized phải liên tục đọc trọng số và thực hiện dequantization; sau khi đủ
worker để giữ pipeline bận, thêm threads không tạo thêm băng thông bộ nhớ. Ngược lại,
chúng cạnh tranh cache/băng thông dùng chung và tăng chi phí scheduling, đồng bộ.

Điều này thể hiện rõ khi tăng từ 2 lên 8 và 16 threads: throughput giảm lần lượt từ
9.0 xuống 7.9 và 7.6 tok/s. `ngl=99` cũng offload phần lớn layer qua Vulkan nên khả
năng scale bằng CPU threads càng hạn chế. Vì vậy `-t 2` là knee hợp lý; speedup 1.18×
không lớn nhưng là before/after đo được trên cùng model, quantization và máy.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
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
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
