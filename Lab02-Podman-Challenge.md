# Lab 02 — Challenge: "Máy chủ Docker bị xâm phạm"

> **Thời gian ước lượng: ~60 phút**
> **Đầu vào:** `baseline/` + image `smartapp:1.0` (từ Lab 01)
> **Đầu ra:** một Pod rootless + `smartapp.kube.yaml` — dùng làm đầu vào cho Lab 03.

---

## Bối cảnh

Socket của Docker daemon bị lộ ra ngoài và máy chủ đã bị gỡ khỏi mạng để điều tra. Không còn Docker daemon để dùng. Phải đưa **smartapp** sang chạy bằng **Podman rootless** ngay trong hôm nay — kèm một container **sidecar** thu log — và vẫn phục vụ trên **cổng 80**.

Nhiệm vụ của nhóm: chạy lại smartapp ở chế độ daemonless + rootless, gom nó cùng sidecar vào một Pod, rồi xuất ra manifest Kubernetes để sẵn sàng lên cluster sau này.

## Thông tin đang có

- `baseline/` và image `smartapp:1.0` từ Lab 01.
- Một image nhỏ bất kỳ để làm sidecar thu log (ví dụ `alpine`).

## Mục tiêu

1. **Rootless.** Đưa `smartapp:1.0` vào Podman và chạy ở chế độ rootless (`podman info` báo rootless = `true`). Không bật lại Docker, không chạy bằng root.
2. **Cổng đặc quyền.** Phục vụ trên **cổng 80** ở chế độ rootless — tự xử lý rào cản cổng < 1024.
3. **Pod chia sẻ namespace.** Tạo một Pod gồm **2 container** (smartapp + sidecar) **chia sẻ network namespace** (sidecar gọi được smartapp qua `localhost`).
4. **Bắc cầu Kubernetes.** Xuất Pod thành manifest chuẩn K8s: `smartapp.kube.yaml` (phải chạy lại được bằng `podman play kube`).

## Bàn giao — thư mục `submission/`

```
submission/
  smartapp.kube.yaml
```

Manifest phải dựng lại được toàn bộ Pod (2 container, phục vụ cổng 80) bằng `podman play kube`.

## Tự chấm

```bash
./graders/grade02.sh submission
```

Lặp lại tới khi **mọi mục PASS**. Grader chỉ kiểm trạng thái cuối — nhóm làm cách nào cũng được.

## Ràng buộc

- Không bật lại Docker; không chạy container bằng root.
- Cổng 80 phải hoạt động ở chế độ rootless.
- `smartapp.kube.yaml` phải `podman play kube` được (grader sẽ dùng đúng lệnh đó để dựng lại Pod).

---

### Input cho Lab 03

Giữ lại image `smartapp:1.0`. Ở Lab 03, hệ thống giám sát phát hiện image trên registry **không khớp chữ ký** đã ký tuần trước — nhóm sẽ phải tìm ra image bị tráo, quét SBOM và dựng pipeline ký/xác minh.
