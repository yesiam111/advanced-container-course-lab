# Lab 05 — Challenge: "App đã hardening nhưng chết trong production — và không có shell"

> **Thời gian ước lượng: ~75 phút**
> **Đầu vào:** `smartapp:hardened` (từ Lab 04) + 3 sự cố dựng sẵn
> **Đầu ra:** các container đã sửa + `answers.env` (nguyên nhân gốc).

---

## Bối cảnh

`smartapp:hardened` (distroless, **không có shell**) đang chạy lỗi trong production. Vì không exec được shell, nhóm phải gỡ lỗi ở **cấp runtime** — ephemeral debug container, `nsenter`, exit code. Có **ba sự cố** dựng sẵn (giống nhau cho mọi nhóm): một OOM, một CrashLoop, một lỗi mạng.

## Thông tin đang có

Ba container đang lỗi: `lab05-oom`, `lab05-crash`, `lab05-net`.

## Mục tiêu

1. **OOM.** Chẩn đoán `lab05-oom`: tìm tín hiệu thoát và xác nhận OOM. Khắc phục thành một container chạy **ổn định** tên **`lab05-oom-fixed`** (không còn OOMKilled).
2. **CrashLoop.** Chẩn đoán `lab05-crash` từ log + exit code; sửa thành container chạy ổn định tên **`lab05-crash-fixed`**.
3. **Mạng.** Chẩn đoán `lab05-net`: xác định lỗi nằm ở **tầng nào** (DNS / port / process).
4. **Ghi nguyên nhân gốc** vào `answers.env`.

## `answers.env` (trong thư mục `triage/`)

```env
OOM_SIGNAL=...        # tín hiệu/exit code của OOM
CRASH_REASON="..."    # vì sao container crash
NET_LAYER=...         # dns | port | process
```

## Bàn giao — thư mục `triage/`

```
triage/
  answers.env
```
Đồng thời để **đang chạy**: `lab05-oom-fixed`, `lab05-crash-fixed`.

## Tự chấm

```bash
./graders/grade05.sh triage
```

> Grader đọc ground-truth từ file **chỉ giảng viên có** — script an toàn để phát cho nhóm.

## Ràng buộc

- Không "exec shell" vào image hardened (không có shell) — dùng công cụ cấp runtime.
- Hai container `*-fixed` phải đang chạy khi chấm.
