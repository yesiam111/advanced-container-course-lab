# Lab 02 — Podman: Container không cần Daemon

> Ứng dụng xuyên suốt: **smartapp**.
> Lab này: chạy lại smartapp bằng **Podman** ở chế độ **rootless**, build không cần daemon, và đóng gói thành **Podman Pod**.

**Thời lượng:** ~60 phút · **Yêu cầu:** Linux có Podman ≥ 4.x (user thường, không sudo cho phần rootless).

---

## 1. Cài đặt & xác nhận rootless

> **Lưu ý phiên bản:** lab cần **Podman ≥ 4.4** (cho `pasta` & `Quadlet`). Đối với ubuntu 22.04, dùng hướng dẫn sau

Dự án `podman-static` đóng gói sẵn podman + crun + conmon + netavark + slirp4netns + fuse-overlayfs.
```bash
sudo apt-get install -y uidmap                       # cho rootless (newuidmap/newgidmap)
VER=5.5.2                                             # đổi theo tag mới nhất ở trang Releases
curl -fsSL -o podman-linux-amd64.tar.gz \
  https://github.com/mgoltzsche/podman-static/releases/download/v${VER}/podman-linux-amd64.tar.gz
tar -xzf podman-linux-amd64.tar.gz
sudo cp -r podman-linux-amd64/usr podman-linux-amd64/etc /
grep -q "^$(id -un):" /etc/subuid || sudo sh -c "echo $(id -un):100000:65536 >> /etc/subuid"
grep -q "^$(id -un):" /etc/subgid || sudo sh -c "echo $(id -un):100000:65536 >> /etc/subgid"
podman version                                        # 5.x — có pasta + Quadlet
```
> Trang Releases: github.com/mgoltzsche/podman-static/releases (chọn tag mới nhất cho `VER`).

---

## 2. Chạy smartapp rootless và quan sát UID mapping

```bash
podman build -t smartapp:1.0 ./smartapp        # podman build = Buildah engine, không daemon
podman run -d --name s -p 8080:8080 smartapp:1.0
curl -s localhost:8080
```
**Kết quả mong đợi:** `smartapp on <hostname> (pid 1)`.

Chứng minh **không chạy root trên host**:
```bash
ps -o user,pid,cmd -C python        # tiến trình thuộc user thường, KHÔNG phải root
podman top s huser hpid user pid    # so sánh UID trong container vs trên host
cat /proc/self/uid_map ; podman unshare cat /proc/self/uid_map
```
**Quan sát:** root (UID 0) bên trong container ánh xạ sang một UID không đặc quyền trên host (user namespace).

Cleanup
```
podman stop s
podman rm s
```

---

## 3. Mạng rootless (giới hạn cần biết)

```bash
# port < 1024 sẽ lỗi khi rootless:
podman run --rm -p 80:8080 smartapp:1.0 || echo ">> bị chặn: port < 1024 cần cấu hình thêm"
```
**Fix (tùy chọn):** `sudo sysctl net.ipv4.ip_unprivileged_port_start=80` rồi thử lại.
**Quan sát:** mạng rootless chạy ở user-space (slirp4netns/pasta) — đây là đánh đổi so với bridge của root.

---

## 4. Podman Pod — nhóm container chia sẻ namespace

```bash
podman rm -f s 2>/dev/null            # giải phóng cổng 8080 đang bị container ở mục 2 giữ
podman pod create --name smartpod -p 8080:8080
podman run -d --pod smartpod --name web smartapp:1.0
podman run -d --pod smartpod --name sidecar alpine tail -f /dev/null
podman ps -a --pod                    # 'web' phải ở trạng thái Up
podman logs web | tail                # phải thấy: Running on http://0.0.0.0:8080
```
Chứng minh **chia sẻ network namespace** (cùng loopback) — chờ web sẵn sàng trước khi gọi:
```bash
sleep 2
podman exec sidecar wget -qO- 127.0.0.1:8080
```
**Kết quả mong đợi:** sidecar gọi được smartapp qua loopback — vì cùng một network namespace (giống Pod của Kubernetes).

> **Vì sao dùng `127.0.0.1` chứ không `localhost`?** Trong image, `localhost` phân giải ra cả `::1` (IPv6) lẫn `127.0.0.1`. busybox `wget` thử `::1` trước, nhưng Flask dev server chỉ lắng nghe IPv4 → "Connection refused". Dùng `127.0.0.1` (IPv4) là chắc chắn. Đây là một bẫy IPv4/IPv6 rất hay gặp, không phải lỗi mạng của pod.

> Nếu vẫn **Connection refused**: (0) đang gọi `localhost` thay vì `127.0.0.1`? → bẫy IPv6 ở trên, dùng `127.0.0.1`; (1) `podman logs web` xem web có chạy/lắng nghe 0.0.0.0:8080 không; (2) cổng 8080 trên host còn bị container khác (mục 2) giữ → `podman rm -f s` rồi tạo lại pod, hoặc đổi sang `-p 8081:8080`; (3) web chưa kịp khởi động → chờ thêm vài giây; (4) `podman ps --pod` xác nhận web và sidecar **cùng một pod** (sidecar cũ ở pod khác sẽ không thấy web).
> Nếu lỗi **"can only create exec sessions on running containers"**: sidecar đã thoát. Dùng `podman ps -a` / `podman logs sidecar` để xem lý do — thường do lệnh giữ container không hợp lệ. Dùng `tail -f /dev/null` (như trên) thay cho `sleep 1d` để container luôn chạy.

---

## 5. Liên hệ đến Kubernetes

```bash
podman generate kube smartpod > smartpod.yaml
head -30 smartpod.yaml
```
**Quan sát:** manifest YAML chuẩn K8s — có thể `podman play kube smartpod.yaml` cục bộ, hoặc mang lên cluster sau này.

---

## Break it / Fix it

1. Quên `--pod`, sidecar không thấy app qua localhost:
   ```bash
   podman run -d --name lonely alpine tail -f /dev/null
   podman exec lonely wget -qO- localhost:8080 || echo ">> không kết nối: khác network namespace"
   ```
   **Fix:** thêm container vào cùng pod (`--pod smartpod`).

---

## Suy nghĩ
- Vì sao mô hình daemonless + rootless giảm bề mặt tấn công so với Docker daemon chạy root?
- Podman Pod liên hệ thế nào với Pod trong Kubernetes?

## Dọn dẹp
```bash
podman pod rm -f smartpod ; podman rm -f s lonely 2>/dev/null ; rm -f smartpod.yaml
```
