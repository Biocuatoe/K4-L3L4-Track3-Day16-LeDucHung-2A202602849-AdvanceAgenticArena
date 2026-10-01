# Day 16 — Agent Arena (Đấu trường Agent)

Cuộc thi 120 phút tại lớp · Track 3 · VinUniversity

> **Đọc theo thứ tự:** `README.md` (trang này — bức tranh tổng thể) → [`GUIDE.md`](GUIDE.md)
> (hướng dẫn làm từng bước) → [`RUBRIC.md`](RUBRIC.md) (cách chấm điểm chi tiết) →
> [`phases/README.md`](phases/README.md) (luyện tập khác chấm điểm thế nào).

---

## 1. Bạn sẽ làm gì?

Repo có sẵn một tác tử (agent) hoạt động theo cơ chế ReAct (Suy luận + Hành động: Reasoning + Acting) chạy được nhưng **cố tình yếu**. Nó mắc năm lỗi:

| # | Lỗi của agent yếu | Hậu quả |
|---|---|---|
| 1 | **Bịa đặt** (hallucinate) số liệu khi tài liệu không có | mất điểm trung thực (honesty) |
| 2 | **Trích dẫn sai** (misattribute) tài liệu (câu thật, nguồn sai) | mất điểm bám chứng cứ (grounding) |
| 3 | **Nghe lời tài liệu độc** (tấn công chèn lệnh – prompt injection) | mất điểm an toàn (safety) |
| 4 | **Tiêu quá ngân sách** (budget) gọi công cụ (tool) | mất điểm hiệu quả (efficiency) |
| 5 | **Không nhận ra** khi công cụ trả về rác (degraded output) | trả lời bằng tài liệu chưa từng đọc |

Việc của bạn: viết **5 lớp bảo vệ (layer)** gắn vào **6 điểm móc can thiệp (hook)** có sẵn của khung điều khiển
(harness) theo kiến trúc phần mềm trung gian (middleware). Bạn **không** viết lại agent, **không** viết lại prompt — bạn chỉ **bọc** nó lại.

```
        ┌────────────────── khung điều khiển (harness) của BẠN ──────────────────┐
 brief ─►  injection_guard · critic · citation_checker · budget_policy · retry   ─► báo cáo (report)
        │                         bọc quanh                                      │
        │                  agent ReAct (baseline, yếu)                           │
        └─────────────────────────────────────────────────────────────────────────┘
```

Bạn được chấm trên ba tiêu chí: **bám chứng cứ** (grounding), **an toàn** (safety), **hiệu quả/tiết kiệm** (efficiency).

---

## 2. Từ điển nhanh

| Thuật ngữ | Nghĩa dễ hiểu |
|---|---|
| **Tác tử** (agent) | Chương trình dùng mô hình ngôn ngữ lớn (LLM) để tự lập kế hoạch, gọi công cụ và trả lời |
| **Mô hình ReAct** (Reasoning + Acting) | Kiểu agent hoạt động lặp: *suy nghĩ (reason) → gọi công cụ (act) → quan sát kết quả (observe) → suy nghĩ tiếp* |
| **Khung điều khiển** (harness) | Lớp vỏ bọc quanh agent để điều phối, kiểm soát luồng chạy, đo đếm tài nguyên và xử lý lỗi |
| **Phần mềm trung gian** (middleware) | Mô hình “vỏ củ hành” (onion): nhiều lớp xếp chồng, mỗi lớp được chặn hoặc biến đổi luồng vào/ra |
| **Điểm móc can thiệp** (hook) | Vị trí cố định trong vòng đời của agent cho phép các lớp middleware can thiệp vào |
| **Đề bài nghiên cứu** (brief) | Nhiệm vụ gồm câu hỏi, ngân sách (budget), dữ kiện cần thiết (required facts) và đáp án chuẩn (ẩn) |
| **Kho tài liệu** (corpus) | Tập hợp văn bản đóng vai trò tri thức ngoài để agent tra cứu (`data/corpus/*.json`) |
| **Khẳng định** (claim) | Một mệnh đề cụ thể do agent đưa ra, kèm mã `doc_id` của tài liệu làm bằng chứng |
| **Danh sách trích dẫn** (citations) | Danh sách `doc_id` agent đã dùng — chỉ để tham khảo, **không** tính điểm trực tiếp |
| **Bám chứng cứ** (grounding) | Tiêu chí đánh giá: mọi khẳng định đưa ra đều phải có căn cứ xác thực từ tài liệu trong kho |
| **Ảo giác / bịa đặt** (hallucination) | Khẳng định do mô hình tự nghĩ ra mà không có căn cứ trong tài liệu nào của kho |
| **Từ chối trả lời** (abstain) | Chủ động thông báo “không đủ căn cứ” thay vì phỏng đoán hay bịa đặt khi thiếu dữ liệu hoặc mâu thuẫn |
| **Tấn công chèn lệnh** (prompt injection) | Kỹ thuật chèn lệnh giả mạo vào nội dung tài liệu nhằm thao túng hành vi của agent |
| **Chuỗi bẫy nhận diện** (canary string) | Chuỗi ký tự đặc biệt giấu trong tài liệu độc; nếu lọt vào báo cáo nghĩa là agent đã bị thao túng |
| **Nguồn gốc dữ liệu** (provenance) | Tính nguyên bản: claim phải do chính mô hình viết ra và là trích dẫn nguyên văn một dòng trong tài liệu |
| **Cổng kiểm định** (gate) | Điều kiện tiên quyết ĐẠT/TRƯỢT; nếu trượt cổng trace thì toàn bộ bài thi nhận 0 điểm |
| **Vết thực thi** (trace) | Tệp nhật ký JSONL ghi nhận tuần tự mọi sự kiện diễn ra trong suốt lượt chạy |
| **Mô hình giả lập** (mock model) | Mô hình giả chạy ngoại tuyến (offline), kết quả cố định, dùng để phát triển và kiểm thử nhanh |

---

## 3. Lịch 120 phút

| Phút | Việc | Ghi chú |
|---|---|---|
| **0 – 15** | **Làm quen** | Chạy thử, đọc `arena/scorer.py`, mở 5 file layer |
| **15 – 95** | **Xây dựng** | 80 phút này *là* cả bài lab. Viết 5 layer, chạy lại, đo |
| **95 – 105** | **Đóng băng & nộp** | Ngừng sửa `harness/`; `git commit` + `git push` |
| **105 – 120** | **Vòng chấm điểm** | Giảng viên chạy, bạn ngồi xem — không sửa gì nữa |

### 15 phút đầu — chạy đúng các lệnh sau

```bash
cd Day16-AgentArena-Student
python3 -m pytest -q                              # môi trường ổn chưa?
python3 scripts/run_practice.py --layers none     # agent yếu, chưa có layer nào
```

Lệnh thứ hai chạy dưới 2 giây (mô hình giả, offline, **không cần API key**) và in bảng điểm.
Điểm trung bình khoảng **24/100** là điểm xuất phát của mọi người. Một bộ 5 layer hoàn chỉnh đạt
khoảng **81.71** trên đúng bộ đề này. Khoảng cách đó chính là bài lab.

Yêu cầu: Python 3.12+, `pip install -r requirements.txt` (chỉ có `pytest`). Không cần mạng.
Có thể kiểm tra sâu hơn bằng `python3 scripts/verify.py` (~20 giây).

---

## 4. Cái nào của bạn, cái nào đóng băng?

### ✅ `harness/` là của bạn — sửa thoải mái

| File | Vai trò |
|---|---|
| `harness/middleware.py` | 6 hook, lớp cơ sở `Middleware`, và ví dụ `LoggingMiddleware` dùng đủ 6 hook |
| `harness/agent.py` | Agent ReAct baseline |
| `harness/layers/*.py` | **5 file bạn phải điền** |

### 🚫 `arena/` bị đóng băng — chỉ đọc, không sửa

Bạn **được phép và nên đọc** `arena/scorer.py` (luật chơi công khai, giống nhau cho mọi người).
Nhưng **sửa bất kỳ file nào trong `arena/` là huỷ bài thi**: vòng chấm điểm kiểm tra mã băm
(hash) của thư mục này.

### ⚠️ Ba thứ trong `harness/` tuy của bạn nhưng đừng đụng nếu không có lý do chính đáng

1. **`MAX_STEPS = 40`** trong `agent.py` — hạ thấp thì agent có thể hết bước trước khi ra `FINAL`,
   không có báo cáo, điểm 0 mà **không báo lỗi nào**.
2. **`arena.model.parse_output`** — đừng thay bằng bộ phân tích “dễ tính” của riêng bạn. Nó dựng được
   báo cáo đẹp từ đoạn text mà bộ chấm không công nhận → mọi claim bị chấm `NOT_FROM_MODEL`
   (đo được: 40.15 thay vì 92.52).
3. **Không bọc `try/except` quanh code hook.** Layer raise lỗi thì cả lượt chạy chết, bài về 0.
   Cố ý như vậy: nuốt lỗi âm thầm còn tệ hơn gãy to lúc luyện tập.

---

## 5. Năm layer phải viết

Mỗi file trong `harness/layers/` có docstring dài nói rõ **lỗi cần sửa**, **tín hiệu phát hiện**,
và **bẫy đã đo được**. **Hãy đọc docstring trước khi viết dòng nào** — nó trả lời gần hết câu hỏi.

Phần TODO mỗi file chỉ **10–25 dòng**. Một người review độc lập đã cài đủ 5 layer với thân hàm
6, 6, 13, 15 và 22 dòng — tổng cộng 62 dòng.

| Layer | Sửa lỗi gì | Hook chính | Điểm kiếm được |
|---|---|---|---|
| `critic` (phản biện / tự đánh giá - reflection & self-critique) | Mô hình không bao giờ nói “không biết” — nó bịa. Xoá claim không có căn cứ; từ chối trả lời (abstain) khi không còn gì. **Kiếm nhiều điểm nhất.** | `after_agent` | trung thực (honesty) + chính xác (precision) |
| `citation_checker` (kiểm tra trích dẫn - citation verification) | Câu thì thật, nguồn thì sai. Gắn lại mỗi claim về đúng tài liệu chứa nó trong kho đã đọc. | `after_agent` | bám chứng cứ (grounding) |
| `injection_guard` (phòng vệ chèn lệnh - prompt injection defense) | Coi nội dung tài liệu là **dữ liệu**, không phải **mệnh lệnh**. Cách ly đoạn độc, quét sạch chuỗi bẫy (canary) ở câu trả lời (`answer`). | `wrap_tool_call` + `after_agent` | 15 điểm an toàn (safety injection) |
| `budget_policy` (chính sách ngân sách - budget policy) | Kế hoạch mô hình luôn dài 11 lượt, 4 lượt cuối là rác. Ép chốt kết luận (`FINAL`) khi hết ngân sách. | `before_model` + `wrap_tool_call` | hiệu quả (efficiency) |
| `retry` (thử lại công cụ - tool retry) | Công cụ hỏng ngẫu nhiên (~15%). Thử lại ở *dưới* mô hình để không tốn lượt suy luận. | `wrap_tool_call` | giảm độ dao động / phương sai (variance) |

Hai điều cần biết trước:

- **`scripts/run_practice.py` tự cài 5 layer đúng thứ tự.** Bạn chỉ cần điền phần TODO.
- **`Doc.tags` luôn rỗng** qua `ctx.corpus` (cả luyện tập lẫn chấm điểm). Các nhãn bẫy như
  `outdated`, `contradiction`, `injection` bị gỡ ngay khi runner dựng corpus. Layer dựa vào `tags`
  sẽ về 0 đúng lúc quan trọng nhất. (File trên đĩa `data/corpus/*.json` ở vòng luyện tập vẫn còn nhãn.)

---

## 6. Sáu hook (điểm móc can thiệp)

Mỗi hook mặc định không làm gì (no-op); bạn chỉ ghi đè (override) phương thức nào cần dùng.

```text
    before_agent(ctx)  [chạy 1 lần khi bắt đầu]
    ┌─ Vòng lặp từng lượt (ReAct loop) ─────────────────────────┐
    │  messages = before_model(ctx, messages)                   │
    │  ┌ wrap_model_call(ctx, call, messages) ───────────────┐  │
    │  │      response = model.complete(messages)            │  │
    │  └─────────────────────────────────────────────────────┘  │
    │  (runner tự ghi sự kiện model_call vào vết chạy trace)    │
    │  response = after_model(ctx, response)                    │
    │  nếu gặp FINAL -> kết thúc vòng lặp                       │
    │  ┌ wrap_tool_call(ctx, call, name, args) ──────────────┐  │
    │  │      result = tools.<name>(**args)                  │  │
    │  └─────────────────────────────────────────────────────┘  │
    └───────────────────────────────────────────────────────────┘
    report = after_agent(ctx, report)  [chạy 1 lần sau khi lặp xong]
    tools.submit(report)               # Nộp báo cáo chính thức để chấm
```

| Hook | Chạy khi nào | Mục đích tiêu biểu |
|---|---|---|
| `before_agent(ctx)` | **Một lần**, trước khi vào vòng lặp | Khởi tạo trạng thái trong `ctx.state`, đọc cấu hình `budget` |
| `before_model(ctx, messages)` | **Mỗi lượt**, trên đường **ra** mô hình | Nhắc chốt câu trả lời (`budget_policy` gửi `NUDGE`) |
| `wrap_model_call(ctx, call, messages)` | **Mỗi lượt**, **bọc quanh** lời gọi mô hình | Giám sát, đếm token hoặc can thiệp gọi lại cấp mô hình |
| `after_model(ctx, response)` | **Mỗi lượt**, trên đường **về** từ mô hình | Kiểm tra câu trả lời thô trước khi đưa vào lịch sử agent |
| `wrap_tool_call(ctx, call, name, args)` | **Mỗi lượt gọi công cụ**, bọc quanh công cụ | Lọc đoạn độc (`injection_guard`), gọi lại khi lỗi (`retry`) |
| `after_agent(ctx, report)` | **Một lần**, sau vòng lặp và **trước** `tools.submit` | Lọc bịa đặt (`critic`), chỉnh trích dẫn (`citation_checker`) |

### Thứ tự chạy khi danh sách `middleware = [A, B, C]`

* **Chạy xuôi (A → B → C):** `before_agent`, `before_model`. Lớp đứng sau nhận đầu ra của lớp đứng trước.
* **Lồng nhau kiểu củ hành (A bọc ngoài B, B bọc ngoài C):** `wrap_model_call`, `wrap_tool_call`. A chạy đầu tiên và nhận hàm `call` chính là B bọc quanh C. Nếu một lớp không gọi `call(...)`, các lớp bên trong sẽ bị chặn hoàn toàn.
* **Chạy ngược (C → B → A):** `after_model`, `after_agent`. Lớp đứng đầu danh sách (`A`) sẽ là lớp xử lý **cuối cùng** trên đường ra.

> **Thứ tự chuẩn của 5 lớp:** `[injection_guard, critic, citation_checker, budget_policy, retry]`.
> Vì `injection_guard` đứng đầu danh sách nên phương thức `after_agent` của nó sẽ chạy sau cùng để rà soát sạch chuỗi bẫy (canary) trong câu trả lời cuối cùng (`answer`).

---

## 7. Hai luật “im lặng mà đắt” — nhớ kỹ

Cả hai đã được **đo thật**, đều thất bại **không báo lỗi**, chỉ điểm tụt.

### 7.1. Claim phải là trích dẫn **nguyên văn** của **một dòng**

Một claim chỉ được tính điểm khi thoả **cả ba** điều kiện:

1. Là chữ **mô hình thật sự đã viết** (không thì bị chấm `NOT_FROM_MODEL`).
2. Có trong báo cáo đã `submit()` (không thì `NOT_SUBMITTED`).
3. Là bản sao **nguyên văn một DÒNG** trong tài liệu được trích.

Nghĩa là: **diễn đạt lại (paraphrase) không tính · cắt vắt qua hai dòng không tính · thêm một dấu
chấm cuối câu cũng không tính** · đổi nháy cong thành nháy thẳng, “chuẩn hoá” khoảng trắng cũng không.

> Thêm đúng một dấu chấm cuối mỗi claim: **92.52 → 45.36** (mất 47.16 điểm).

Ngoại lệ hợp lệ: **cắt bớt (trim)** — substring vẫn là một trích dẫn. (Cắt còn 120 ký tự mất 8.11 điểm
do giảm recall, nhưng không mất nguồn gốc.) **Cắt thì được, sửa thì không.**

### 7.2. Layer nào viết lại chữ của claim là phá nguồn gốc (provenance)

| Được phép | Layer điển hình |
|---|---|
| Đổi `claim["doc_id"]` (gắn lại nguồn) | `citation_checker` |
| Xoá hẳn claim, hoặc đặt `abstain` | `critic` |
| Cắt bớt `claim["text"]` (substring) | bất kỳ |
| Viết lại `report["answer"]` — **miễn phí** | `injection_guard` |

> **Quy tắc giữa lúc căng thẳng: đổi nguồn, hoặc bỏ claim — đừng bao giờ đổi chữ.**

Dễ vấp nhất ở `injection_guard`: đừng “làm sạch” claim. Làm sạch `answer` là miễn phí;
làm sạch claim thì mất nguồn gốc và mất luôn điểm grounding — đắt hơn nhiều con canary bạn định gỡ.

---

## 8. Công cụ luyện tập

```bash
python3 scripts/run_practice.py                          # cả 9 brief công khai, đủ 5 layer
python3 scripts/run_practice.py --layers none            # baseline (không layer nào)
python3 scripts/run_practice.py --layers critic          # chỉ bật một layer
python3 scripts/run_practice.py --layers critic,citation_checker
python3 scripts/run_practice.py --brief pub-01-sla-hien-hanh   # soi một brief
python3 scripts/run_practice.py --no-flaky               # tắt lỗi ngẫu nhiên (CHỈ để gỡ lỗi)
python3 scripts/run_practice.py --entry ten-doi --out runs/ten-doi.json

python3 scripts/selfeval.py                              # VÌ SAO bạn được đúng ngần ấy điểm
python3 scripts/leaderboard.py runs/*.json               # so sánh nhiều lần chạy
```

Mỗi brief in một dòng: `G / S / E` = grounding / safety / efficiency.
Ba thứ cần để mắt:

1. **`⚠ Không có FINAL đọc được ở: …`** — mọi claim ở brief đó bị chấm `NOT_FROM_MODEL`. Đắt và im
   lặng nhất; sửa trước mọi thứ khác.
2. **`gate_passed` / `gate_reason`** trong `runs/practice.json`. `false` nghĩa là điểm 0.
3. **G tăng mà S tụt** (hoặc ngược lại) — bạn đang đổi chiều này lấy chiều kia. Bật từng layer để tìm thủ phạm.

### `selfeval.py` — chẩn đoán điểm

`run_practice.py` chỉ nói “G 6.9/55”. `selfeval.py` nói *vì sao*: thiếu dữ kiện, diễn đạt lại thay vì
trích nguyên văn, trích sai tài liệu, hay claim mô hình chưa từng viết. Mỗi brief in 6 khối:
G/S/E + cổng trace · bẫy của brief (DÍNH hay TRÁNH ĐƯỢC) · dữ kiện bắt buộc (✓ trích đúng, ~ nói mà không
trích, ✗ thiếu) · từng claim đã nộp · an toàn · **SỬA GÌ TRƯỚC**.

Dòng quý nhất là **`SUÝT ĐÚNG`**: chỉ thẳng ký tự bạn lỡ thêm/bớt (ví dụ “CHỈ LỆCH DẤU CÂU tại ký tự
thứ 175”) — dấu hiệu của §7.2: một layer của bạn đã viết lại `claim["text"]`.

`selfeval.py` **không** chạy trên bộ brief có tính điểm và **không** đổi điểm của bạn.

---

## 9. Nộp bài và vòng chấm điểm

**Phút 95 — đóng băng.** Ngừng sửa `harness/`, rồi:

```bash
git add -A
git commit -m "Agent Arena — <tên đội>"
git push
```

Thứ được thu là **thư mục `harness/`**. Điểm **không** lấy từ `runs/*.json` bạn đẩy lên, mà từ lượt
chạy do giảng viên thực hiện.

**Phút 105–120 — chấm điểm.** Giảng viên chạy layer của bạn dưới **runner đóng băng**, với **mô hình
thật**, trên **bộ brief riêng** bạn chưa từng thấy, cùng corpus nhưng đã gỡ nhãn bẫy. Hệ quả:

1. **Không hard-code brief** (không `if brief_id == ...`, không danh sách `doc_id` ăn may).
2. **Không dựa vào `Doc.tags`.**
3. **Mô hình thật viết khác mô hình giả:** thụt lề, in đậm, bọc code fence, viết thường, thêm câu kết.
   Layer giả định output có đúng một hình dạng cố định sẽ vỡ ở đây và *chỉ* ở đây.

---

## 10. Điểm luyện tập chỉ để tham khảo

Bộ brief công khai **để gỡ lỗi, không để xếp hạng**. Một harness 30 dòng “trích dòng dài nhất của top-5
tài liệu” đạt **87.30** ở luyện tập nhưng chỉ **47.40** ở vòng chấm.

Câu hỏi đúng của vòng luyện tập: **“5 layer của tôi có thật sự hoạt động không?”** Cách kiểm tra tốt nhất
là **leave-one-out** (bỏ từng cái): rút một layer khỏi stack đầy đủ, xem điểm có tụt không.

```bash
python3 scripts/run_practice.py --layers injection_guard,critic,citation_checker,budget_policy   # bỏ retry
```

Rút `retry` mà điểm không đổi → `retry` của bạn chưa làm gì. (Lưu ý: giá trị thật của `retry` là giảm
**phương sai** — độ lệch chuẩn từ 24.21 xuống 11.43 — chứ không phải tăng trung bình. Giảm một nửa dao
động đáng giá hơn một chút điểm trung bình: đó là khác biệt giữa bài chắc chắn và bài may rủi.)

> Điểm thật đến từ một lượt chạy duy nhất, do giảng viên thực hiện, trên brief bạn chưa từng đọc.
> **Hãy build cho lượt chạy đó.**
