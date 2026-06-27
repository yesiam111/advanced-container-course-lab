# Lab 01 — Challenge: "Kế thừa một image bí ẩn"

> **Thời gian ước lượng: ~75 phút**
> **Đầu vào:** (không có — đây là challenge đầu tiên)
> **Đầu ra:** `baseline/` (image gọn + `answers.env`) — dùng làm đầu vào cho Lab 02.

---

## Bối cảnh

Một đồng nghiệp vừa nghỉ việc và để lại **smartapp** dưới dạng **một file image duy nhất** — `smartapp-legacy.tar`. Không có Dockerfile, không ghi chú, không ai nhớ image được build thế nào. Image **quá nặng** và **bị chết** khi chạy trên các node nhỏ của team.

Nhiệm vụ: tìm hiểu image này đang có vấn đề gì, rồi **tạo một baseline image sạch và gọn** để dùng trong challenge tiếp theo.

## Thông tin đang có

- `lab01-workdir/smartapp-legacy.tar`

Không có mã nguồn, không có Dockerfile — phải tự khôi phục những gì cần từ chính image được cung cấp.

---

## Mục tiêu

1. **Image mới, nhỏ hơn nhiều, GIỮ NGUYÊN hành vi.** Build một image mới (đặt tên `smartapp:1.0`) phục vụ đúng như cũ: `GET /` trả về dòng `smartapp ...` và `GET /health` trả về `ok`, ở cổng **8080**.
2. **Kích thước < 120 MB.** (Image gốc nặng hơn ngưỡng này rất nhiều — đây là phần việc chính.)
3. **Khôi phục nội tại của image legacy bằng tay** và điền `answers.env`:
   - `CMD` — lệnh mà image legacy chạy khi khởi động.
   - `LAYERS` — số **layer filesystem** của image legacy.
   - `MIN_MEM_MB` — mức `--memory` **nhỏ nhất** (đơn vị MB) mà **image MỚI của nhóm** vẫn khởi động và phục vụ được.

> Grader kiểm `MIN_MEM_MB` bằng cách chạy thật image của nhóm: phải **phục vụ được ở `MIN_MEM_MB`** và **bị OOM ở `MIN_MEM_MB − 16`**. Nói cách khác, nhóm phải tìm đúng ngưỡng (sai số 16 MB), không đoán bừa.

## `answers.env` (đặt trong thư mục `baseline/`)

```env
CMD="..."
LAYERS=...
MIN_MEM_MB=...
```

> Lưu ý: đặt giá trị trong dấu nháy kép nếu có dấu cách (ví dụ `CMD="python app.py"`).

## Output — thư mục `baseline/`

```
baseline/
  Dockerfile        (hoặc Containerfile)
  app.py
  requirements.txt
  answers.env
```

Image build ra từ `baseline/` phải đặt tên **`smartapp:1.0`** — Lab 02 sẽ dùng lại image này.

## Tự chấm

```bash
./graders/grade01.sh baseline
```

Lặp lại tới khi **mọi mục REQUIRED đều PASS**.

## Ràng buộc & gợi ý mức cao (không phải lời giải)

- **không có** Dockerfile, chỉ có file tarball.
- "Giữ nguyên hành vi": cùng route, cùng cổng, cùng nội dung trả về — không cần giữ cùng base image hay cùng kích thước.
- Mọi dữ kiện cần thiết đều **nằm trong chính image legacy**.

---

### Input cho Lab 02

Giữ lại thư mục `baseline/` và image `smartapp:1.0`. Ở Lab 02, máy chủ Docker bị gỡ khỏi mạng sau một sự cố lộ socket — sẽ phải mang chính `smartapp:1.0` này sang chạy **rootless bằng Podman**.
