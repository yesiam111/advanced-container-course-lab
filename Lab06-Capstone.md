# Lab 06 — Capstone: Bảo mật & Vận hành smartapp End-to-End

> Bài tập tổng hợp toàn khóa. Mục tiêu: đưa **smartapp** từ một image "chạy được" thành image **đạt chuẩn production** — gọn, an toàn, có nguồn gốc tin cậy và quan sát được.

**Thời lượng:** 2–3 giờ (cá nhân hoặc nhóm) · **Yêu cầu:** toàn bộ công cụ của Lab 01–05.

---

## 0. Chuẩn bị môi trường — Ubuntu 22.04 server MỚI

> Nếu capstone chạy trên một server 22.04 hoàn toàn mới (chưa có gì), cài đủ bộ công cụ dưới đây. Nếu đã làm Lab 01–05 trên cùng máy thì đa số đã có — bỏ qua phần trùng.

```bash
# --- Docker Engine (kho chính thức) ---
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg python3 jq
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER     # đăng xuất/đăng nhập lại để chạy docker không cần sudo

# --- Trivy (apt) ---
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy

# --- Syft + Cosign (đủ cho capstone: SBOM + ký) ---
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sudo sh -s -- -b /usr/local/bin
curl -fsSL -o cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo install -m 0755 cosign /usr/local/bin/cosign && rm -f cosign

# --- Registry cục bộ để push/sign/verify offline (không cần tài khoản Docker Hub) ---
docker run -d -p 5000:5000 --restart=always --name reg registry:2
```
Xác nhận: `docker version ; trivy --version ; syft version ; cosign version ; python3 --version`.

> **Lưu ý:** grader (`grade.sh`) chạy trên **máy của giảng viên** (cần `docker`, `trivy`, `cosign`, `python3`, `curl`) — học viên chỉ cần tạo ra thư mục bài nộp trên server của mình. Phần "rootless/cap-drop/seccomp/read-only" được chấm qua cấu hình `docker run` trong `run.sh`, nên Docker là đủ (không bắt buộc Podman cho capstone).

---

## Điểm xuất phát: biến thể riêng từng học viên

Mỗi học viên **không** bắt đầu từ smartapp sạch, mà nhận một **biến thể lỗi & không an toàn duy nhất** (sinh bằng `capstone-grader/make-variant.sh <id>`): app có token riêng + một bug `/compute` riêng, một dependency lỗ hổng riêng, Dockerfile/run.sh không an toàn. Nhiệm vụ là **chẩn đoán → sửa → hardening** chính biến thể của mình.

> Vì token, công thức `/compute` và dependency khác nhau giữa các học viên, **không thể copy-paste lời giải mẫu hay bài bạn khác** — grader sẽ phát hiện ngay.

## Cách nộp & chấm (tự động)

Nộp **trạng thái cuối** dưới dạng một thư mục được chấm tự động bằng `capstone-grader/grade.sh <bài-nộp> keys/<id>.key` (xem `capstone-grader/README.md`).

Thư mục bài nộp gồm:
```
Containerfile  app.py  requirements.txt  seccomp.json
run.sh  sbom.json  pipeline.yml  [COSIGN_IMAGE.txt]
```
- Grader tự build thành image `smartapp:grade`, chạy `run.sh` (container `smartapp-grade`, cổng 8080), rồi kiểm tra tự động.
- Chấm điểm bằng: `./grade.sh <thư-mục-bài-nộp>` → in bảng điểm; **đạt khi ≥ 80/100**.

Điểm chỉ phụ thuộc kết quả cuối có kiểm chứng được bằng máy — sinh viên tập trung làm đúng, không phải làm đẹp báo cáo.

## Đề bài

Hoàn thành 5 bước dưới đây trên smartapp. Mọi yêu cầu đều phản ánh trong các tiêu chí mà grader kiểm tra (xem bảng điểm).

### Bước 1 — Build & Inspect (Bài 1)
- Viết **multi-stage** Containerfile cho ra image **distroless, nonroot**.
- Liệt kê layer, chỉ ra digest của config, so sánh kích thước với `smartapp:1.0`.
- ✅ *Đạt:* image hardened nhỏ hơn rõ rệt; giải thích được vai trò từng layer.

### Bước 2 — Run an toàn (Bài 2 + 4)
- Chạy **rootless** (Podman) với `--cap-drop ALL`, seccomp, `--read-only`, `no-new-privileges`.
- Chứng minh không chạy root trên host (uid_map) và app vẫn phục vụ `:8080`.
- ✅ *Đạt:* app hoạt động dưới mọi ràng buộc trên; chứng minh được từng ràng buộc.

### Bước 3 — Supply chain (Bài 3)
- Quét CVE (Trivy), tạo **SBOM** (Syft), **ký** (Cosign), và dựng pipeline `scan → sign → push → verify` tự dừng khi có CVE CRITICAL.
- ✅ *Đạt:* `cosign verify` thành công; pipeline fail đúng lúc khi inject CVE.

### Bước 4 — Tấn công & Phòng thủ (Bài 4)
- Tái hiện **một** kịch bản escape (vd `--privileged` + mount `/`) trong VM lab.
- Áp dụng phòng chống và chứng minh đã **chặn được**.
- ✅ *Đạt:* trình bày được trước/sau và cơ chế phòng chống.

### Bước 5 — Vận hành (Bài 5)
- Bật metrics (docker stats/cAdvisor) + thu log.
- Dựng **một** sự cố (OOM hoặc CrashLoop) và chẩn đoán theo quy trình: Triệu chứng → Thu thập → Khoanh vùng → Nguyên nhân → Khắc phục.
- ✅ *Đạt:* xác định đúng nguyên nhân gốc và khắc phục, có bằng chứng (exit code/metrics/log).

---

## Bảng điểm tự động (100)

| Nhóm | Điểm | Grader kiểm tra |
|------|------|-----------------|
| Image | 25 | build được · nonroot · không shell · multi-stage · size < 250MB |
| Bảo mật runtime | 30 | read-only · cap-drop ALL · no-new-privileges · seccomp · không privileged/không mount nhạy cảm |
| Chuỗi cung ứng | 20 | SBOM hợp lệ · Trivy không CVE CRITICAL · cosign verify |
| Quan sát | 10 | có HEALTHCHECK · /health phản hồi |
| Biến thể & chức năng | 15 | "/" trả về token đúng · `/compute` đã sửa (N×FACTOR) · đã xử lý dependency lỗ hổng |

> Hai loại tiêu chí: (a) **kỹ năng** (image/runtime/supply/observability) — áp dụng đúng các control; (b) **biến thể** (15đ) — chứng minh đã thực sự làm trên biến thể của mình (rebuild từ source được giao, sửa đúng bug, xử lý đúng dependency). Phòng thủ chấm gián tiếp qua cấu hình an toàn (không privileged, không mount nhạy cảm).
