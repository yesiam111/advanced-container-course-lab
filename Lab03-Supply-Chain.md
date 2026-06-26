# Lab 03 — Supply Chain Security

> Ứng dụng xuyên suốt: **smartapp**.
> Lab này: quét CVE, tạo **SBOM**, **ký** image với Cosign keyless, kiểm tra từ xa bằng Skopeo, và ghép thành pipeline **scan → sign → push → verify**.

**Thời lượng:** ~75 phút · **Yêu cầu:** Docker/Podman, `trivy`, `syft`, `grype`, `cosign`, `skopeo`, một registry (Docker Hub hoặc registry local).

---

## 0. Cài đặt công cụ (Ubuntu)

```bash
# Trivy (Aqua Security) — qua repo apt:
sudo apt-get install -y wget gnupg
wget -qO- https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy

# Syft & Grype (Anchore) — script cài chính thức:
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh  | sudo sh -s -- -b /usr/local/bin
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sudo sh -s -- -b /usr/local/bin

# Cosign (Sigstore) — tải binary mới nhất:
curl -fsSL -o cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo install -m 0755 cosign /usr/local/bin/cosign && rm -f cosign

# Skopeo — có sẵn trong kho Ubuntu:
sudo apt-get install -y skopeo
```
Xác nhận:
```bash
echo " --- Trivy ---" ; trivy --version
echo " --- Syft ---"; syft version
echo " --- Grype ---"; grype version
echo " --- Cosign ---"; cosign version
echo " --- Skopeo ---"; skopeo --version
```
> Tải lần đầu, Trivy/Grype sẽ cập nhật cơ sở dữ liệu lỗ hổng (~vài trăm MB) — cần Internet.

---

## 1. Quét lỗ hổng với Trivy

```bash
trivy image smartapp:1.0 | tee trivy-report.txt
trivy image --severity HIGH,CRITICAL smartapp:1.0
```
**Kết quả mong đợi:** vì base `python:3.9-slim` đã cũ, Trivy liệt kê nhiều CVE (HIGH/CRITICAL) trong OS packages và `werkzeug/flask`.

---

## 2. Tạo SBOM với Syft, quét CVE từ SBOM với Grype

```bash
syft smartapp:1.0 -o cyclonedx-json > sbom.json
syft smartapp:1.0 -o table | head -20          # xem thành phần & phiên bản
grype sbom:sbom.json | tee grype-report.txt
```
**Quan sát:** SBOM liệt kê toàn bộ thành phần; Grype phát hiện CVE **từ SBOM** mà không cần build lại — tạo một lần, quét nhiều lần.

---

## 3. Đẩy lên registry + ký với Cosign

> **Chọn registry.** Lab chạy được với Docker Hub *hoặc* một registry cục bộ.
> - **Registry cục bộ (khuyến nghị cho lớp — không cần tài khoản):**
>   ```bash
>   docker run -d -p 5000:5000 --name reg registry:2
>   export IMG=localhost:5000/smartapp:1.0
>   ```
>   Với registry cục bộ không-TLS, thêm `--tls-verify=false` cho `skopeo`/`cosign` (vd `cosign sign --allow-insecure-registry`).
> - **Docker Hub:** `export IMG=docker.io/<user>/smartapp:1.0`, rồi đăng nhập **cho cả docker, podman/skopeo**:
>   ```bash
>   docker login docker.io        # dùng Personal Access Token (PAT), KHÔNG dùng mật khẩu
>   skopeo login docker.io        # skopeo/podman có auth file riêng — phải login riêng
>   ```
>   Lỗi `unauthorized: incorrect username or password` ở skopeo = chưa `skopeo login` (hoặc PAT sai).

> **Ký & xác minh theo DIGEST, không theo tag.** Tag (vd `:1.0`) có thể bị đẩy đè bằng image khác → ký/verify theo tag sẽ xác nhận nhầm. Digest `sha256:...` là content-addressable, bất biến — đúng nội dung đã ký (chính là phòng chống registry tampering).

```bash
export IMG=docker.io/<user>/smartapp:1.0   # đổi <user>
docker tag smartapp:1.0 $IMG && docker push $IMG

# Lấy DIGEST của image vừa push (repo@sha256:...):
export REF=$(docker inspect --format '{{index .RepoDigests 0}}' "$IMG")
echo "$REF"                  # vd docker.io/<user>/smartapp@sha256:abc123...
```

**Cách A — Key-based (khuyến nghị cho VM headless: KHÔNG mở browser):**
```bash
export COSIGN_PASSWORD="lab"            # demo: khỏi hỏi mật khẩu (production dùng secret thật)
cosign generate-key-pair               # -> cosign.key (riêng) + cosign.pub (công khai)
cosign sign   --key cosign.key --yes "$REF"                 # ký theo DIGEST, không OIDC/browser
cosign verify --key cosign.pub "$REF" | python3 -m json.tool | head -20
```

**Cách B — Keyless (Sigstore):** cần OIDC. Trên máy có browser sẽ mở trình duyệt; trong CI (GitHub Actions...) dùng token OIDC sẵn có nên KHÔNG cần browser.
```bash
cosign sign --yes "$REF"               # mở browser để xác thực OIDC (máy có GUI)
cosign verify "$REF" \
  --certificate-identity-regexp ".*" \
  --certificate-oidc-issuer-regexp ".*" | python3 -m json.tool | head -20
# Production keyless: ghim chính xác --certificate-identity "you@example.com" --certificate-oidc-issuer "https://github.com/login/oauth"
```
**Kết quả mong đợi:** `cosign verify` theo digest xác nhận đúng image đã ký, không thể bị tráo qua tag.

(Tùy chọn) gắn SBOM như attestation (cũng theo digest):
```bash
cosign attest --yes --predicate sbom.json --type cyclonedx "$REF"
```

---

## 4. Skopeo — kiểm tra từ xa, không cần pull

```bash
skopeo login docker.io 
skopeo inspect docker://$IMG | python3 -m json.tool | head -30
skopeo inspect docker://$IMG --format '{{.Digest}}'    # lấy DIGEST từ registry mà không cần pull
skopeo inspect --raw docker://$IMG | python3 -m json.tool | head    # manifest + digest
# copy sang registry khác / air-gapped:
# skopeo copy docker://$IMG oci-archive:smartapp.oci
```
**Quan sát:** lấy được manifest, **digest**, danh sách layer **mà không pull** image về máy — dùng digest này để ghim/verify thay vì tin tag.

---

## 5. Pipeline scan → sign → push → verify (script)

`pipeline.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail
IMG=docker.io/<user>/smartapp:ci

echo "[1/4] BUILD"     ; docker build -t "$IMG" ./smartapp
echo "[2/4] SCAN"      ; trivy image --exit-code 1 --severity CRITICAL "$IMG" || { echo "CRITICAL CVE -> DỪNG"; exit 1; }
echo "[3/4] PUSH+SIGN" ; docker push "$IMG"
REF=$(docker inspect --format '{{index .RepoDigests 0}}' "$IMG")     # ghim DIGEST
cosign sign --yes "$REF"
echo "[4/4] VERIFY"    ; cosign verify "$REF" --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*" >/dev/null && echo "OK -> $REF"
```
**Kết quả mong đợi:** nếu có CVE CRITICAL, pipeline **dừng ở bước SCAN** (exit 1) — đúng tinh thần "fail the build".

---

## 6. (Nâng cao) Pipeline trong CI thật (GitHub Actions)

`.github/workflows/supply-chain.yml`
```yaml
name: supply-chain
on: [push]
jobs:
  build-scan-sign:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write, id-token: write }
    steps:
      - uses: actions/checkout@v4
      - uses: aquasecurity/trivy-action@master
        with: { image-ref: 'ghcr.io/${{ github.repository }}:ci', severity: 'CRITICAL', exit-code: '1' }
      - uses: anchore/sbom-action@v0
        with: { image: 'ghcr.io/${{ github.repository }}:ci' }
      - id: build
        run: |
          docker build -t ghcr.io/${{ github.repository }}:ci ./smartapp
          docker push ghcr.io/${{ github.repository }}:ci
          echo "ref=$(docker inspect --format '{{index .RepoDigests 0}}' ghcr.io/${{ github.repository }}:ci)" >> "$GITHUB_OUTPUT"
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ${{ steps.build.outputs.ref }}    # ký theo DIGEST
```
**Mục tiêu:** thấy đúng các bước scan → SBOM → sign chạy tự động, dùng OIDC của CI cho keyless (không cần khóa).

---

## 7. (Nâng cao) Provenance attestation

```bash
cosign attest --yes --predicate sbom.json --type cyclonedx "$REF"
cosign verify-attestation --type cyclonedx "$REF" --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*" | python3 -m json.tool | head
```
**Quan sát:** attestation gắn "image này được build từ gì" vào chính image — nền tảng cho SLSA.

---

## 8. (Nâng cao) Private registry với xác thực

```bash
docker run -d -p 5000:5000 --name reg registry:2
export IMG2=localhost:5000/smartapp:1.0
docker tag smartapp:1.0 $IMG2 && docker push $IMG2
skopeo inspect docker://$IMG2 --tls-verify=false | python3 -m json.tool | head
```
**Thảo luận:** bật authentication + TLS; chỉ chấp nhận image đã ký (content trust); mô hình mirror & air-gapped.

---

## 9. (Nâng cao) Bắt một dependency độc hại được cài cắm

```bash
# thêm một thư viện cũ/nhiều lỗ hổng vào requirements rồi build lại:
echo "pyyaml==5.1" >> smartapp/requirements.txt
docker build -t smartapp:bad ./smartapp
grype smartapp:bad | grep -i pyyaml   # Grype chỉ ra CVE của phiên bản đó
```
**Mục tiêu:** thấy SBOM + scan phát hiện ngay thành phần nguy hiểm. **Fix:** ghim phiên bản an toàn rồi build lại.

---

## Break it / Fix it

1. **Đẩy đè image khác lên cùng tag (registry tampering):**
   ```bash
   # tạo một image khác rồi push đè CÙNG tag :1.0 (mô phỏng kẻ tấn công tráo image)
   echo "RUN echo tampered" >> smartapp/Dockerfile
   docker build -t $IMG ./smartapp && docker push $IMG
   NEWREF=$(docker inspect --format '{{index .RepoDigests 0}}' "$IMG")
   echo "tag cũ:$REF  ->  tag giờ trỏ:$NEWREF"     # digest đã đổi!

   cosign verify "$REF"    --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*" >/dev/null && echo ">> DIGEST đã ký: VẪN hợp lệ (an toàn)"
   cosign verify "$NEWREF" --certificate-identity-regexp ".*" --certificate-oidc-issuer-regexp ".*" || echo ">> image mới ở tag: KHÔNG có chữ ký (đã chặn được)"
   ```
   **Bài học:** verify theo **digest** (`$REF`) vẫn đúng image đã ký; nếu tin **tag**, bạn sẽ nhận image bị tráo. → Luôn pull/deploy theo digest.
2. **CVE CRITICAL chặn pipeline** → **Fix:** chuyển sang base mới hơn (xem Lab 04 multi-stage/distroless) rồi chạy lại.

---

## Suy ngẫm
- Vì sao tách SBOM khỏi việc quét lại có giá trị khi có CVE mới (vd XZ backdoor)?
- Bước verify ở cấp cluster (admission) khác gì verify trong CI? (→ khóa Kubernetes)

## Dọn dẹp
```bash
rm -f sbom.json trivy-report.txt grype-report.txt pipeline.sh
```
