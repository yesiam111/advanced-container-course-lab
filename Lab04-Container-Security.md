# Lab 04 — Container Security Nâng cao

> Ứng dụng xuyên suốt: **smartapp**.
> Lab này: hardening smartapp (multi-stage → distroless, cap-drop, seccomp, read-only), tái hiện một **container escape có kiểm soát**, và phát hiện bằng **Falco**.

**Thời lượng:** ~90 phút · **Yêu cầu:** Docker/Podman, `strace`, `jq`, AppArmor utils, (tùy chọn) `falco`. **Chạy trong VM/lab cô lập** cho phần escape.

---

## 0. Cài đặt công cụ (Ubuntu)

```bash
# Công cụ cơ bản:
sudo apt-get update
sudo apt-get install -y strace jq libcap2-bin apparmor-utils
#   strace: thu thập syscall  · jq: đọc JSON  · libcap2-bin: capsh/getpcaps  · apparmor-utils: aa-status, apparmor_parser

# Falco (tùy chọn — phần phát hiện runtime) — repo apt chính thức:
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/falcosecurity.list
sudo apt-get update && sudo apt-get install -y falco
#   Khi cài, chọn driver "Modern eBPF" (không cần kernel headers; hợp VM kernel 5.8+).
```
Xác nhận:
```bash
strace --version | head -1 ; jq --version ; aa-status --version 2>/dev/null || apparmor_parser --version
sudo falco --version 2>/dev/null || echo "(falco tùy chọn — bỏ qua nếu không cài)"
```
> AppArmor đã bật sẵn trên Ubuntu (`sudo aa-status`). SELinux KHÔNG dùng trên Ubuntu — phần SELinux trong bài chỉ so sánh khái niệm.

---

## 1. Capability tối thiểu

```bash
docker run -d --name s1 smartapp:1.0
docker exec s1 sh -c 'cat /proc/1/status | grep CapEff'   # xem capability đang có
# chạy lại với quyền tối thiểu:
docker run -d --name s_min --cap-drop ALL --cap-add NET_BIND_SERVICE smartapp:1.0
docker exec s_min sh -c 'id; cat /proc/1/status | grep CapEff'
```
**Quan sát:** `--cap-drop ALL` xóa toàn bộ capability; chỉ thêm lại cái cần. Tránh `CAP_SYS_ADMIN`.

---

## 2. Tạo & áp dụng seccomp profile tùy chỉnh

Bắt đầu từ profile mặc định của Docker, hoặc tạo profile chặn `ptrace`/`mount`:

`seccomp.json` (rút gọn — chặn theo default, mặc định deny là tùy chọn nâng cao)
```json
{ "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [ { "names": ["ptrace","mount","reboot"], "action": "SCMP_ACT_ERRNO" } ] }
```
```bash
docker run --rm --security-opt seccomp=seccomp.json smartapp:1.0 &
docker run --rm --security-opt seccomp=seccomp.json smartapp:1.0 sh -c 'mount -t tmpfs none /mnt' \
  || echo ">> mount bị seccomp chặn (đúng mong đợi)"
```
**Mẹo nâng cao:** dùng `strace -f -c` để thu thập syscall thực sự app dùng rồi sinh profile allow-list tối thiểu.

---

## 3. Image hardening: multi-stage → distroless, nonroot, read-only

`Dockerfile.hardened`
```dockerfile
# Build base PHẢI cùng phiên bản Python với distroless runtime (python3-debian12 = 3.11),
# nếu không gói cài vào path python3.9 sẽ không được runtime 3.11 nạp -> ModuleNotFoundError.
FROM python:3.11-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=build /app/deps /app/deps
COPY app.py .
ENV PYTHONPATH=/app/deps          # để Python tìm thấy gói đã cài
EXPOSE 8080
USER nonroot
CMD ["app.py"]
```
```bash
docker build -t smartapp:hardened -f smartapp/Dockerfile.hardened ./smartapp
docker images smartapp                       # so sánh kích thước 1.0 vs hardened
trivy image --severity HIGH,CRITICAL smartapp:hardened   # ít CVE hơn hẳn
docker run -d --name sh --read-only --cap-drop ALL \
  --security-opt no-new-privileges --tmpfs /tmp smartapp:hardened
docker exec sh sh 2>/dev/null || echo ">> không có shell (distroless) — bề mặt tấn công nhỏ"
```
**Kết quả mong đợi:** image nhỏ hơn, ít CVE hơn, không shell, chạy nonroot + read-only.

---

## 4. Container escape có kiểm soát (LAB CÔ LẬP)

```bash
# kịch bản rủi ro: privileged + mount host root
docker run --rm -it --privileged -v /:/host alpine sh -c \
  'ls /host/etc/shadow >/dev/null 2>&1 && echo ">> ĐỌC ĐƯỢC /etc/shadow CỦA HOST = escape"'
```
**Phòng chống — chạy lại an toàn:**
```bash
docker run --rm -it --cap-drop ALL alpine sh -c \
  'ls /host 2>/dev/null || echo ">> không truy cập được host (đã an toàn)"'
```
**Quan sát:** escape đến từ **cấu hình sai** (`--privileged`, mount `/`), không phải lỗi runtime.

Thử thêm **hai kịch bản escape khác** (lab cô lập), kèm phòng chống:
```bash
# B) mount Docker socket → điều khiển daemon từ trong container
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock docker sh -c \
  'docker ps >/dev/null 2>&1 && echo ">> điều khiển được Docker host = escape"'
# Phòng chống: KHÔNG mount docker.sock.

# C) writable hostPath nhạy cảm
docker run --rm -v /etc:/hostetc alpine sh -c 'ls /hostetc/shadow && echo ">> đọc được /etc của host"'
# Phòng chống: không mount thư mục host nhạy cảm; dùng :ro.
```

---

## 5. Phát hiện bất thường với Falco (tùy chọn)

```bash
# Bật đúng unit theo driver đã chọn lúc cài (modern eBPF là phổ biến nhất):
sudo systemctl enable --now falco-modern-bpf      # hoặc falco-bpf / falco-kmod
sudo systemctl status falco-modern-bpf --no-pager | head
# Xem alert thời gian thực ở TERMINAL 1:
sudo journalctl -u falco-modern-bpf -f
```
```bash
# TERMINAL 2 — kích hoạt hành vi đáng ngờ: mở shell trong container
docker run --name trig -d busybox tail -f /dev/null && docker exec -it trig sh -c 'id'
# -> TERMINAL 1 hiện: "A shell was spawned in a container ..."
docker rm -f trig
```

**(Tùy chọn) ghi alert ra file** — sửa `/etc/falco/falco.yaml`:
```yaml
file_output:
  enabled: true
  filename: /var/log/falco.log
```
```bash
sudo systemctl restart falco-modern-bpf
sudo tail -f /var/log/falco.log
```

Viết rule tùy chỉnh trong `/etc/falco/falco_rules.local.yaml` để cảnh báo khi có tiến trình bất thường trong smartapp.

`falco_rules.local.yaml` (ví dụ)
```yaml
- rule: Shell in smartapp
  desc: Phát hiện shell chạy trong container smartapp
  condition: spawned_process and container and proc.name in (sh, bash) and container.image.repository contains "smartapp"
  output: "Shell trong smartapp (user=%user.name cmd=%proc.cmdline)"
  priority: WARNING
```
Kích hoạt nhiều hành vi để thấy alert: mở shell, ghi vào `/etc`, đọc `/etc/shadow`.

---

## 6. (Nâng cao) Seccomp profile từ strace

```bash
# thu thập syscall thực sự dùng:
strace -f -qcf -o syscalls.txt python smartapp/app.py & sleep 5; kill %1
awk 'NR>2 {print $NF}' syscalls.txt | sort -u > used-syscalls.txt
head used-syscalls.txt
```
Sinh profile allow-list từ danh sách trên (mỗi syscall một entry `SCMP_ACT_ALLOW`, default `SCMP_ACT_ERRNO`), rồi chạy `--security-opt seccomp=profile.json`.
**Mục tiêu:** một profile tối thiểu thực sự cho smartapp; thử bỏ một syscall cần thiết để thấy app lỗi → hiểu đánh đổi.

---

## 7. (Nâng cao) AppArmor profile trên Ubuntu

`/etc/apparmor.d/smartapp`
```
#include <tunables/global>
profile smartapp flags=(attach_disconnected) {
  #include <abstractions/base>
  deny /etc/shadow r,
  network,
}
```
```bash
sudo apparmor_parser -r /etc/apparmor.d/smartapp
docker run --rm --security-opt apparmor=smartapp alpine cat /etc/shadow || echo ">> bị AppArmor chặn"
```
**Quan sát:** MAC chặn truy cập ngoài chính sách, bất kể quyền file thông thường.

## Suy ngẫm
- "Defense in depth" gồm những lớp nào ở đây, và lớp nào cứu bạn nếu một lớp thất bại?
- Các thiết lập này map sang `securityContext` của Kubernetes ra sao? (→ khóa K8s)

## Dọn dẹp
```bash
docker rm -f s1 s_min sh 2>/dev/null ; rm -f seccomp.json
```
