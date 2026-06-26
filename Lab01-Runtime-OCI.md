# Lab 01 — Container Runtime & Chuẩn OCI

> Ứng dụng xuyên suốt: **smartapp** — một web service Python/Flask đóng gói trong một container.
> Mục tiêu: build image của smartapp, soi cấu trúc image theo chuẩn OCI, và chạy nó **trực tiếp ở cấp runtime** (không qua Docker CLI).

**Thời lượng:** ~60 phút · **Yêu cầu:** Linux có Docker + containerd (`crictl`, `ctr`), quyền sudo. Có thể cài nhanh docker với lệnh `curl -fsSL https://get.docker.com | sh`.
  - Lưu ý không dùng lệnh trên để cài docker daemon trên môi trường production do có rủi ro về bảo mật hoặc tương thích

---
## 0. Mã nguồn smartapp

Tạo thư mục `smartapp/` với 3 file sau (dùng lại cho mọi lab).

`app.py`
```python
from flask import Flask
import socket, os
app = Flask(__name__)

@app.route("/")
def home():
    return f"smartapp on {socket.gethostname()} (pid {os.getpid()})\n"

@app.route("/health")
def health():
    return "ok\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

`requirements.txt`
```
Flask==3.1.2
```

`Dockerfile`
```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
```

---

## 1. Build image và soi cấu trúc OCI

```bash
docker build -t smartapp:1.0 ./smartapp
docker image inspect smartapp:1.0 --format '{{.Id}}'
docker history smartapp:1.0
```
**Kết quả mong đợi:** mỗi chỉ thị Dockerfile tạo một layer; `docker history` liệt kê các layer theo thứ tự.

Xuất image ra archive và xem cấu trúc:
```bash
docker save smartapp:1.0 -o smartapp.tar
rm -rf smartapp_oci && mkdir smartapp_oci && tar -xf smartapp.tar -C smartapp_oci
ls smartapp_oci
```
**Tùy phiên bản Docker, có 2 định dạng:**
- **OCI layout** (Docker mới / containerd image store): có `index.json`, `oci-layout`, `blobs/sha256/...` — *đây chính là OCI Image Spec đã học*.
  ```bash
  python3 -m json.tool smartapp_oci/index.json
  ```
- **docker-archive cũ** (graphdriver overlay2): có `manifest.json` ở gốc.
  ```bash
  python3 -m json.tool smartapp_oci/manifest.json
  ```
**Quan sát:** cả hai đều trỏ tới một `Config` (sha256:...) và danh sách `Layers` theo digest — content-addressable, đúng OCI Image Spec.

---

## 2. Quan sát overlay filesystem (copy-on-write)

> Lưu ý storage store: kiểm tra driver trước.
> ```bash
> docker info --format '{{.Driver}}'
> ```
> - `overlay2` → Docker dùng **graphdriver cổ điển**, có trường `.GraphDriver` (cách A).
> - `overlayfs` → Docker dùng **containerd image store (snapshotter)** mặc định ở Docker mới; `.GraphDriver` rỗng → dùng **cách B** (đọc bảng mount của kernel, không phụ thuộc store).

```bash
docker run -d --name s1 smartapp:1.0
```

**Cách A — graphdriver cổ điển (driver = overlay2):**
```bash
docker inspect s1 --format '{{ json .GraphDriver }}' | python3 -m json.tool
sudo ls -l "$(docker inspect s1 --format '{{ .GraphDriver.Data.UpperDir }}')"
```

**Cách B — không phụ thuộc store đang sử dụng (mọi store, kể cả containerd `overlayfs`):**
Đọc trực tiếp lớp overlay từ bảng mount của kernel:
```bash
# từ trong container:
docker exec s1 cat /proc/mounts | awk '$2=="/"'
# hoặc từ host:
PID=$(docker inspect -f '{{.State.Pid}}' s1)
sudo awk '$5=="/"' /proc/$PID/mountinfo
```
**Quan sát:** dòng mount `overlay` chứa `lowerdir=` (các layer R/O dùng chung) và `upperdir=` / `workdir=` (lớp ghi R/W riêng của container).

Kiểm chứng copy-on-write — cách rõ ràng & không phụ thuộc store là `docker diff`:
```bash
docker exec s1 sh -c 'echo hello > /tmp/test.txt'
docker diff s1     # liệt kê file Added/Changed/Deleted ở LỚP GHI so với image
```
**Kết quả mong đợi:** thấy `A /tmp/test.txt` (và `C /tmp`) — thay đổi được ghi vào lớp R/W của container, các layer image (R/O) không đổi.

(Tùy chọn) Xem file thật nằm trong `upperdir` trên host (mọi store):
```bash
PID=$(docker inspect -f '{{.State.Pid}}' s1)
UPPER=$(sudo awk '$5=="/"{print $NF}' /proc/$PID/mountinfo | grep -o 'upperdir=[^,]*' | cut -d= -f2)
echo "upperdir = $UPPER"
sudo ls -l "$UPPER/tmp/test.txt"
```

---

## 3. Chạy container trực tiếp bằng containerd (không Docker)

```bash
sudo systemctl status containerd --no-pager | head -3
sudo ctr images pull docker.io/library/nginx:latest
sudo ctr images ls | grep nginx
sudo ctr run -d docker.io/library/nginx:latest web1
sudo ctr containers ls
sudo ctr task ls
```
**Kết quả mong đợi:** container `web1` chạy mà **không cần Docker daemon** — chứng minh containerd hoạt động độc lập.

Dọn dẹp:
```bash
sudo ctr task kill web1 ; sudo ctr tasks rm web1;  sudo ctr containers rm web1
```

---

## 4. Kiểm tra bằng crictl (công cụ chuẩn CRI)

Cài đặt `crictl` và cấu hình containerd CRI plugin

```bash
VERSION=v1.34.0

curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/${VERSION}/crictl-${VERSION}-linux-amd64.tar.gz

sudo tar -C /usr/local/bin -xzf crictl-${VERSION}-linux-amd64.tar.gz

crictl --version


sudo sed -i 's/^disabled_plugins = \["cri"\]/# disabled_plugins = ["cri"]/' /etc/containerd/config.toml
sudo systemctl restart containerd
```


Tạo cấu hình để hết cảnh báo endpoint (khỏi truyền `--runtime-endpoint` mỗi lần):
```bash
sudo tee /etc/crictl.yaml >/dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
EOF
```

Pull và xem image **qua CRI** (image đi vào namespace `k8s.io` của containerd):
```bash
sudo crictl pull nginx:latest
sudo crictl images
sudo crictl ps -a
sudo crictl pods
```

**Vì sao mục 3 (ctr) và Docker không hiện trong crictl?** containerd tách theo *namespace*:
```bash
sudo ctr namespaces ls
sudo ctr -n moby   containers ls   # container của Docker nằm ở namespace moby
sudo ctr -n default images ls      # image pull bằng ctr ở mục 3 nằm ở default
sudo ctr -n k8s.io  images ls      # image crictl vừa pull nằm ở k8s.io
```
- **Docker → `moby`**, **`ctr` mặc định → `default`**, **crictl/CRI → `k8s.io`**. Khác namespace nên không thấy nhau — điểm mấu chốt khi debug trên node Kubernetes.
- **`crictl ps` vẫn trống** trên host thường vì chưa có workload do CRI/kubelet tạo. Trên node Kubernetes thật (hoặc k3s/kind), `crictl ps` và `crictl pods` mới liệt kê đầy đủ pod & container — đó là lúc crictl phát huy tác dụng.

> (Tùy chọn) Cài **k3s** (`curl -sfL https://get.k3s.io | sh -`) rồi chạy lại `sudo crictl ps` để thấy container hệ thống của Kubernetes — đúng trải nghiệm debug ở node.

---

## 5. So sánh cgroup v1 và v2

```bash
stat -fc %T /sys/fs/cgroup/            # cgroup2fs = v2 ; tmpfs = v1
docker run -d --name s2 --memory=64m smartapp:1.0
# tìm cgroup của container theo PID (không phụ thuộc cgroup driver / store):
PID=$(docker inspect -f '{{.State.Pid}}' s2)
CG="/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)"   # cgroup v2
cat "$CG/memory.max"
```
**Kết quả mong đợi:** giá trị `memory.max` ≈ 67108864 (64Mi) — giới hạn cgroup được áp dụng đúng.
> Trên cgroup v1, đường dẫn nằm dưới `/sys/fs/cgroup/memory/...`; cách lấy theo PID ở trên vẫn chỉ đúng controller.

---

## 6. (Nâng cao) Đọc config của image

**Cách nhanh (mọi phiên bản Docker) — qua `docker inspect`:**
```bash
docker inspect smartapp:1.0 -f '{{json .Config}}'  | python3 -m json.tool   # Env, Cmd, Entrypoint, WorkingDir, User
docker inspect smartapp:1.0 -f '{{json .RootFS.Layers}}' | python3 -m json.tool   # diff_ids; đếm = số layer
```

**Cách "bằng tay" từ archive đã giải nén ở mục 1** — script tự đi từ `index.json` → (image index) → manifest → config, hoặc dùng `manifest.json` cũ:
```bash
python3 - <<'PY'
import json, os
base = "smartapp_oci"
def blob(dig): return os.path.join(base, "blobs", "sha256", dig.split(":")[1])
if os.path.exists(os.path.join(base, "index.json")):           # OCI layout (Docker mới)
    obj = json.load(open(os.path.join(base, "index.json")))
    obj = json.load(open(blob(obj["manifests"][0]["digest"]))) # blob đầu có thể là image index (multi-arch)
    while "config" not in obj and "manifests" in obj:          # đi sâu tới manifest thật
        obj = json.load(open(blob(obj["manifests"][0]["digest"])))
    cfg = blob(obj["config"]["digest"])
else:                                                          # docker-archive cũ
    cfg = os.path.join(base, json.load(open(os.path.join(base, "manifest.json")))[0]["Config"])
print(json.dumps(json.load(open(cfg)), indent=2))
PY
```
**Quan sát:** đối chiếu `Env`, `Cmd`, `Entrypoint` với Dockerfile; đếm số `diff_ids` (trong `rootfs`) = số layer.

---

## 7. (Nâng cao) Khám phá layer với `dive`

```bash
# cài dive (https://github.com/wagoodman/dive)

DIVE_VERSION=$(curl -sL "https://api.github.com/repos/wagoodman/dive/releases/latest" | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
curl -fOL "https://github.com/wagoodman/dive/releases/download/v${DIVE_VERSION}/dive_${DIVE_VERSION}_linux_amd64.deb"
sudo apt install ./dive_${DIVE_VERSION}_linux_amd64.deb
```

Dùng dive để phân tích docker image
```bash
dive smartapp:1.0 --ci 2>/dev/null | head -40   # hoặc chạy tương tác: dive smartapp:1.0
```
**Mục tiêu:** thấy mỗi chỉ thị Dockerfile thêm/sửa file gì ở từng layer, phát hiện file thừa làm phình image.

---

## 8. (Nâng cao) Stress test cgroup
```bash
docker run --name stress \
  --memory=128m \
  polinux/stress \
  stress --vm 1 --vm-bytes 200M --timeout 20s

docker inspect stress --format '{{.State.OOMKilled}}'
## true

docker rm stress
```
**Kết quả mong đợi:** container bị OOM khi vm-bytes > memory.


```bash
docker run --rm -d \
  --name stress \
  --memory=128m \
  --cpus=0.5 \
  polinux/stress \
  stress --cpu 2 --timeout 20s
docker stats stress 2>/dev/null
```
**Kết quả mong đợi:** container không vượt quá ~0.5 CPU dù cố "ăn" 2 CPU — cgroup ép đúng giới hạn.

---

## Break it / Fix it

1. Đặt giới hạn quá thấp khiến app không khởi động:
   ```bash
   docker run --rm --memory=8m smartapp:1.0
   ```
   → container bị giết (OOM). **Fix:** tăng `--memory` lên mức hợp lý (vd 128m).
2. `--cpus=0.1` khiến app phản hồi rất chậm. **Fix:** nâng giới hạn CPU hoặc tối ưu app.

---

## Dọn dẹp
```bash
docker rm -f s1 s2 2>/dev/null ; rm -rf smartapp_oci smartapp.tar
```
