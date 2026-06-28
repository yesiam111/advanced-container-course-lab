# Lab 06 — Capstone: Bảo mật & Vận hành smartapp End-to-End

> Bài tập tổng hợp toàn khóa. Mục tiêu: đưa **smartapp** từ một image "chạy được" thành image **đạt chuẩn production** — gọn, an toàn, có nguồn gốc tin cậy và quan sát được.

**Thời lượng:** ~5.5–6.5 giờ (đạt ≥80);
 **Yêu cầu:** toàn bộ công cụ Lab 01–05, **chạy trong VM cô lập** có **Falco đang chạy** (grader đọc file host & bắn cảnh báo).

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

# --- Syft + Cosign (SBOM + ký) ---
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sudo sh -s -- -b /usr/local/bin
curl -fsSL -o cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo install -m 0755 cosign /usr/local/bin/cosign && rm -f cosign

# --- Falco (chấm mục phát hiện runtime) — repo apt chính thức ---
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" | sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt-get update && sudo apt-get install -y falco     # chọn driver "Modern eBPF"

# --- Registry cục bộ để push/sign/verify offline (không cần tài khoản Docker Hub) ---
docker run -d -p 5000:5000 --restart=always --name reg registry:2
```
Xác nhận: `docker version ; trivy --version ; syft version ; cosign version ; sudo falco --version`.

> **Lưu ý:** **chấm trong VM cô lập** vì grader chủ động đọc file host (ablation) và bắn Falco. `grade.sh` cần `docker`, `cosign`, `python3`, `curl`; nên có `trivy`, `syft`, **`falco` đang chạy**. **Cổng cứng:** nếu container chạy privileged hoặc mount nhạy cảm → **TRƯỢT ngay** bất kể điểm.

---

## Điểm xuất phát: biến thể riêng từng học viên

Mỗi học viên **không** bắt đầu từ smartapp sạch, mà nhận một **biến thể lỗi & không an toàn duy nhất** (sinh bằng `capstone-grader/make-variant.sh <id>`): app có token riêng + một bug `/compute` riêng, một dependency lỗ hổng riêng, Dockerfile/run.sh không an toàn. Nhiệm vụ là **chẩn đoán → sửa → hardening** chính biến thể của mình.

> Vì token, công thức `/compute` và dependency khác nhau giữa các học viên, **không thể copy-paste lời giải mẫu hay bài bạn khác** — grader sẽ phát hiện ngay.

## Cách nộp & chấm (tự động)

Nộp **trạng thái cuối** dưới dạng một thư mục được chấm tự động bằng `capstone-grader/grade.sh <bài-nộp> keys/<id>.key` (xem `capstone-grader/README.md`).

Thư mục bài nộp gồm:
```
Containerfile  
app.py  
requirements.txt  
seccomp.json  
run.sh
falco_rule.yaml  
sbom.json  
pipeline.yml  
COSIGN_IMAGE.txt  
cosign.pub
```
- `run.sh` phải chạy **chính image đã ký** theo digest: `docker run "$(cat COSIGN_IMAGE.txt)" ...` (container `smartapp-grade`, cổng 8080). Grader pull image đã ký để soi và đối chiếu.
- Chấm: `./grade.sh <thư-mục-bài-nộp> keys/<id>.key` → bảng điểm; **đạt khi ≥ 80/100 *và* không kích hoạt cổng cứng**.

Điểm chỉ phụ thuộc kết quả cuối có kiểm chứng được bằng máy — sinh viên tập trung làm đúng, không phải làm đẹp báo cáo.

## Đề bài

Hoàn thành 5 bước dưới đây trên smartapp. Mọi yêu cầu đều phản ánh trong các tiêu chí mà grader kiểm tra (xem bảng điểm).

### Bước 1 — Build & Inspect (Bài 1)
- Viết **multi-stage** Containerfile cho ra image **distroless, nonroot**.
- Liệt kê layer, chỉ ra digest của config, so sánh kích thước với `smartapp:1.0`.
- ✅ *Đạt:* image hardened nhỏ hơn rõ rệt; giải thích được vai trò từng layer.

### Bước 2 — Run an toàn (Bài 2 + 4)
- Chạy với `--cap-drop ALL`, `--read-only` (+ `--tmpfs /tmp`), `no-new-privileges`, và **seccomp tự viết** chặn syscall nguy hiểm — phải gồm **`mknod`/`mknodat`** (grader chứng minh bằng EPERM: mặc định gọi được, dưới profile của bạn thì bị chặn).
- App vẫn phục vụ `:8080` dưới mọi ràng buộc.
- ✅ *Đạt:* app hoạt động đủ ràng buộc; seccomp **thực sự** chặn (EPERM); **không** privileged/mount nhạy cảm (vi phạm = cổng cứng).

### Bước 3 — Supply chain + nhất quán (Bài 3)
- Quét CVE (Trivy library), tạo **SBOM** (Syft), **ký theo digest** (Cosign), dựng pipeline `scan → sign → push → verify`.
- **Coherence:** image **CHẠY == ĐÃ KÝ == subject của SBOM** (cùng một digest) — `run.sh` chạy `$(cat COSIGN_IMAGE.txt)`, SBOM tạo từ image đã ký.
- ✅ *Đạt:* `cosign verify` theo digest thành công; ba digest trùng nhau.

### Bước 4 — Tấn công, Phòng thủ & Phát hiện (Bài 4)
- Hardening để chặn escape. **Grader tự kiểm (ablation):** thử bỏ từng control rồi xác nhận tấn công quay lại → control của bạn phải **thực sự** chặn (read-only, cap-drop, không mount host).
- Kèm **`falco_rule.yaml`** bắt hành vi đọc file nhạy cảm; grader sẽ kích hoạt và kiểm cảnh báo.
- ✅ *Đạt:* các rào là thật (ablation pass) và Falco bắn cảnh báo khi đọc `/etc/shadow`.

### Bước 5 — Vận hành (Bài 5)
- Bật metrics (docker stats/cAdvisor) + thu log.
- Dựng **một** sự cố (OOM hoặc CrashLoop) và chẩn đoán theo quy trình: Triệu chứng → Thu thập → Khoanh vùng → Nguyên nhân → Khắc phục.
- ✅ *Đạt:* xác định đúng nguyên nhân gốc và khắc phục, có bằng chứng (exit code/metrics/log).

---

## Bảng điểm tự động (100)

| Nhóm | Điểm | Grader kiểm tra |
|------|------|-----------------|
| Image | 20 | build · multi-stage · nonroot · không shell · size < 250MB |
| Bảo mật runtime | 24 | read-only · cap-drop ALL · no-new-privileges · seccomp CHẶN THẬT (EPERM mknod) · không privileged/mount nhạy cảm |
| Tấn công & phòng thủ (ablation) | 12 | host-mount lộ shadow nhưng cấu hình đã chặn · read-only là rào thật · cap-drop là rào thật |
| Chuỗi cung ứng & nhất quán | 20 | SBOM hợp lệ · Trivy library sạch CRITICAL · cosign verify theo digest · SBOM == digest đã ký · container chạy == digest đã ký |
| Phát hiện runtime (Falco) | 8 | rule hợp lệ · bắn cảnh báo khi đọc shadow |
| Quan sát | 6 | HEALTHCHECK · /health |
| Biến thể & chức năng | 10 | token đúng · `/compute` = N×FACTOR · đã xử lý dependency lỗ hổng |

> **Cổng cứng:** privileged hoặc mount nhạy cảm → **TRƯỢT ngay** bất kể tổng điểm.
> Khác bản cũ: phòng thủ nay chấm **chủ động** (ablation — bỏ control thì tấn công quay lại), thêm **coherence** (chạy==ký==SBOM) và **Falco**; điểm phân bổ lại để vẫn /100.
