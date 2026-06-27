# Lab 04 — Challenge: "Pentest: container đọc được /etc/shadow của host"

> **Thời gian ước lượng: ~90 phút**
> **Đầu vào:** image smartapp (từ Lab 03) + `run.sh.risky` (cấu hình rủi ro)
> **Đầu ra:** `smartapp:hardened` (distroless, siết quyền) — dùng cho Lab 05.

---

## Bối cảnh

Một báo cáo pentest kết luận: container smartapp đang chạy có thể **đọc `/etc/shadow` của host**. 
Nhiệm vụ: tái hiện lỗ hổng, tìm cấu hình nào gây ra, rồi hardening để vừa **chặn** vừa **phát hiện** được.

## Thông tin đang có

- Image smartapp (còn shell) để tái hiện.
- `lab04-workdir/run.sh.risky` — lệnh chạy mà pentest đã dùng.

## Mục tiêu

1. **Tái hiện.** Dùng `run.sh.risky` cho thấy container đọc được `/etc/shadow` của host. Xác định cấu hình nào cho phép điều đó.
2. **Image gọn & an toàn.** Chuyển smartapp sang **multi-stage → distroless**, chạy **nonroot**, **không còn shell**. Đặt tên `smartapp:hardened`.
3. **Chạy tối thiểu quyền.** Viết `run.sh` chạy container tên **`smartapp-hardened`** trên cổng **8080** với: `--cap-drop ALL`, `--read-only`, `--security-opt no-new-privileges`, `--security-opt seccomp=seccomp.json`, **không** `--privileged`, **không** mount nhạy cảm. Sau khi siết, thao tác đọc `/etc/shadow` phải **thất bại**.
4. **Phát hiện.** Viết một rule **Falco** kích hoạt khi có hành vi đọc file nhạy cảm (vd `/etc/shadow`).

## Bàn giao — thư mục `hardened/`

```
hardened/
  Dockerfile (hoặc Containerfile)
  app.py
  requirements.txt
  seccomp.json
  run.sh                 # chạy container 'smartapp-hardened' trên cổng 8080
  falco_rule.yaml
```

## Tự chấm

```bash
./graders/grade04.sh hardened
```

## Ràng buộc

- `run.sh` phải đặt tên container đúng `smartapp-hardened`, cổng 8080 (grader chạy đúng file này rồi `inspect`).
- App vẫn phải phục vụ `/health` sau khi siết quyền.

---

### Input cho Lab 05

Giữ lại `smartapp:hardened` (không có shell). Ở Lab 05, chính image này đang **chết trong production** và nhóm phải gỡ lỗi **mà không exec được shell**.
