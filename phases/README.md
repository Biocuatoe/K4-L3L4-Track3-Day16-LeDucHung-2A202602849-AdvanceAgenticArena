# Hai pha của Agent Arena — Luyện tập và Chấm điểm

Lab chạy theo **hai pha tách biệt**. Chúng khác nhau ở mô hình, bộ đề, người bấm nút chạy — và **chỉ một pha tính điểm thật**. Hiểu sai điều này sẽ dẫn đến việc tối ưu sai hướng, vì vậy hãy đọc kỹ trước khi bắt đầu viết code.

> **Điều hướng:** [`../README.md`](../README.md) (tổng quan) · [`../GUIDE.md`](../GUIDE.md) (hướng dẫn từng bước) · [`../RUBRIC.md`](../RUBRIC.md) (thang điểm chi tiết).

| Tiêu chí | PHA 1 — LUYỆN TẬP (Practice) | PHA 2 — CHẤM ĐIỂM (Scored Round) |
|---|---|---|
| **Mô hình** | `MockModel` (giả lập, ngoại tuyến, tất định - deterministic) | Mô hình **thật**, do giảng viên cấu hình |
| **Bộ đề (brief)** | `data/briefs_public.json` (9 brief công khai) | Bộ đề **riêng** (private), bạn chưa từng thấy |
| **Người thực thi** | Bạn tự chạy bao nhiêu lần tùy ý | **Giảng viên** chạy một lần duy nhất trên máy chấm |
| **Mạng / API key** | Không cần mạng, không tốn API key | Giảng viên thiết lập trên môi trường thi |
| **Điểm số** | Điểm thử nghiệm để gỡ lỗi | **Điểm thi chính thức** |
| **Bảng xếp hạng** | Tự dựng nội bộ bằng `scripts/leaderboard.py` | Bảng xếp hạng chung cuộc của lớp |

---

## Pha 1 — Luyện tập (ngoại tuyến / offline)

```bash
python3 scripts/verify.py        # ~20 giây, kiểm tra toàn bộ môi trường và tính toàn vẹn
python3 scripts/run_practice.py  # chạy agent của bạn trên 9 brief công khai
python3 -m pytest -q             # chạy bộ kiểm thử tự động của harness
```

Vòng luyện tập chạy hoàn toàn **ngoại tuyến trên mô hình giả lập** — đảm bảo tính công bằng: học viên gặp sự cố mạng hay hết hạn ngạch API vẫn có thể lập trình và kiểm thử trơn tru.

**Cách dùng đúng:** kiểm tra xem *5 lớp bảo vệ (layer) của bạn có thực sự hoạt động hay không*, thay vì tìm cách tinh chỉnh ép điểm số trên bộ đề mẫu.

```bash
python3 scripts/run_practice.py --layers none --tag baseline --entry baseline --out runs/baseline.json
python3 scripts/run_practice.py --layers all  --entry me --out runs/me.json
python3 scripts/leaderboard.py runs/ --json
```

Hãy quan sát cột **GAP** — khoảng cách chênh lệch điểm số giữa bài của bạn và mốc cơ sở (baseline) “không có layer nào”. Con số tổng điểm ở pha 1 rất ít ý nghĩa; chỉ số GAP mới phản ánh chính xác các layer bạn viết có mang lại giá trị thật hay không.

### Vì sao điểm luyện tập cao không đồng nghĩa với xếp hạng cao?

Bảy trong chín brief công khai được thiết kế ở mức cơ bản: câu trả lời nằm ngay trong các tài liệu mà câu hỏi gốc truy xuất được ở top-5 kết quả đầu tiên, đúng tại vị trí hàm trích đoạn `arena.model._lift_span` quét qua. Một harness viết vội khoảng 30 dòng — *“tìm kiếm một lần, lấy dòng dài nhất của mỗi kết quả rồi trích dẫn về chính nó”* — có thể đạt điểm rất cao ở pha luyện tập nhưng sẽ rơi về điểm sàn ở vòng chấm điểm thật.

Việc cố tình viết mã cứng (hard-code) dựa trên bộ brief công khai là **hợp lệ nhưng hoàn toàn vô ích**: bộ đề chấm điểm chính thức không dùng chung bất kỳ mã đề (`brief_id`), nội dung câu hỏi hay câu bằng chứng nào.

Riêng hai brief **`pub-08-an-toan-boc-do`** và **`pub-09-so-vu-voi-doi-tac-moi`** được thiết kế sát với phong cách của vòng thi thật: tài liệu chứa câu trả lời **không** nằm trong top kết quả tìm kiếm của câu hỏi ban đầu. Nếu agent chỉ biết đọc thụ động những gì trả về từ lượt tìm kiếm đầu tiên, hai brief này sẽ bộc lộ điểm yếu ngay lập tức. **Hãy coi hai brief này là thước đo sát thực nhất.**

---

## Pha 2 — Vòng chấm điểm (giảng viên chạy)

Dùng **bộ đề riêng trên mô hình thật**. Bộ này **không có** trong bản phát: thư mục `instructor/` không tồn tại,
và `arena.briefs.load_private_briefs()` ném `FileNotFoundError` là **bình thường** trên máy bạn — không phải lỗi.

Mọi thứ trong repo đều được phép đọc. `arena/scorer.py` là bộ chấm thật (không phải bản rút gọn); đọc nó là việc
nên làm, và lab được thiết kế để vẫn công bằng với người đã đọc.

### Bạn nộp gì?

Thư mục **`harness/`**: `agent.py`, `middleware.py`, và 5 layer trong `harness/layers/`.
**Không** nộp `runs/`, **không** sửa `arena/` (phải nguyên vẹn — `scripts/verify.py` kiểm tra MD5), **không** sửa `data/`.

### Được chấm thế nào?

```
tổng = grounding (55) + safety (30) + efficiency (15)
```

Cộng **cổng trace**: ĐẠT/TRƯỢT, không phải chiều điểm thứ tư. Trace không hợp lệ thì cả lượt chạy = 0. Dùng
harness thì cổng này qua miễn phí; tự ghi JSONL hoặc gọi mô hình vòng qua runner thì không. Chi tiết xem
[`../RUBRIC.md`](../RUBRIC.md).

### Pha 2 khác pha 1 thế nào — chuẩn bị trước

- **Mô hình thật không ngoan như mô hình giả.** Nó có thể trả lời ngay lượt đầu không gọi công cụ, in JSON nhiều
  dòng, bọc trong khối mã (code fence), hoặc in đậm nhãn `FINAL:`. `harness/agent.py` đã chuẩn hoá phần lớn các
  dạng đó; `ARENA_SYSTEM_PROMPT_REAL` (tuỳ chọn, xem `--prompt-addendum`) siết thêm giao thức.
- **Brief theo kiểu UNIQUENESS + DEPTH.** Tài liệu chứa đáp án không nằm trong top-k của câu hỏi gốc. Biết **truy
  vấn lại bằng câu hỏi khác** là kỹ năng đang được chấm.
- **Ngân sách chặt hơn.** Mỗi brief có `budget` riêng; efficiency đọc cả số tool call (kể cả `submit`), token và
  thời gian.
- **Nhãn bẫy biến mất.** `Doc.tags` luôn rỗng. Layer nào đọc `doc.tags` để nhận diện tài liệu độc/lỗi thời sẽ im
  lặng ngừng hoạt động. Ở vòng luyện tập, nhãn vẫn còn trong file `data/corpus/*.json` trên đĩa — được nói thẳng ra
  chứ không giấu — nhưng hard-code theo nhãn là cách chắc chắn để mất điểm ở pha 2.

---

## `phases/private/` là gì?

**Không có gì — ở đây.** Đó là nơi giảng viên đặt vật liệu của vòng chấm trên máy của họ, và nó nằm trong
`.gitignore` để một lần `git add -A` bất cẩn không đẩy nó lên. Nếu thư mục này xuất hiện trong bản checkout của
bạn thì có gì đó đã sai — **hãy báo, đừng mở.**

`tests/test_no_instructor_leak.py` canh đúng ranh giới này và chạy cùng bộ test của bạn.
