# Lab 05 — Xử lý Sự cố & Quan sát Container

> Ứng dụng xuyên suốt: **smartapp**.
> Lab này: debug smartapp ở cấp runtime, debug image **distroless không shell** bằng ephemeral container, dùng **nsenter**, và phân tích các sự cố cố ý tạo ra.

**Thời lượng:** ~75 phút · **Yêu cầu:** Docker/Podman, `nsenter`, `crictl`, (tùy chọn) cAdvisor/Prometheus/Grafana/Loki.

---

## 0. Cài đặt công cụ (Ubuntu)

```bash
# nsenter có sẵn trong util-linux (thường đã cài); cài cho chắc:
sudo apt-get update && sudo apt-get install -y util-linux curl
```

---

## 1. Debug cơ bản

```bash
docker run -d --name s1 smartapp:1.0
docker logs s1
docker exec s1 sh -c 'ps aux; ip addr'
docker inspect s1 --format '{{.State.Status}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}}'
```

---

## 2. Debug container distroless (không có shell) bằng ephemeral container

```bash
docker run -d --name sh smartapp:hardened     # distroless, không /bin/sh
docker exec sh sh 2>/dev/null || echo ">> không exec shell được"
# gắn một container debug chia sẻ pid + network namespace:
docker run --rm -it \
  --pid=container:sh --network=container:sh \
  nicolaka/netshoot sh -c 'ps aux; netstat -tlnp; curl -s localhost:8080'
```
**Kết quả mong đợi:** container debug nhìn thấy tiến trình & cổng của smartapp **mà không cần shell** trong image đích, không phải restart nó.
> Trong Kubernetes, đây chính là `kubectl debug` (ephemeral container).

---

## 3. nsenter — vào namespace từ host

```bash
PID=$(docker inspect -f '{{.State.Pid}}' sh)
sudo nsenter -t $PID -n ip addr        # vào NET namespace; chạy 'ip' CỦA HOST -> luôn được
```
Xem **filesystem** của container: với image distroless (không có `ls`/shell), không vào mount namespace mà đọc trực tiếp qua `/proc/<pid>/root` (dùng công cụ của host):
```bash
sudo ls -l /proc/$PID/root/app         # rootfs container nhìn từ host
sudo cat  /proc/$PID/root/app/app.py
```
> Lưu ý: `nsenter -t $PID -m ls /app` sẽ LỖI `failed to execute ls` nếu container không có `ls` (distroless), vì `-m` chuyển sang mount namespace của container. Nếu vẫn muốn vào mount ns, dùng binary có sẵn trong image — distroless có `python3`:
> ```bash
> sudo nsenter -t $PID -m -- python3 -c "import os; print(os.listdir('/app'))"
> ```

**Quan sát:** từ host, "bước vào" namespace mạng của container, và soi filesystem qua `/proc/<pid>/root` — kỹ thuật hiệu quả nhất khi container không có công cụ (distroless).

---

## 4. Phân tích crash: exit code & OOM

```bash
# tạo OOM cố ý:
docker run -d --name oom --memory=16m smartapp:1.0
sleep 3
docker inspect oom --format 'oom={{.State.OOMKilled}} exit={{.State.ExitCode}}'
dmesg | grep -i 'killed process' | tail -2
```
**Kết quả mong đợi:** `oom=true`, `exit=137` (128+SIGKILL 9). dmesg ghi "Killed process".

| Exit | Ý nghĩa |
|------|---------|
| 137  | SIGKILL — thường do OOM |
| 139  | SIGSEGV |
| 143  | SIGTERM (dừng bình thường) |

---

## 5. Quan sát: metrics

```bash
docker stats --no-stream                    # CPU/MEM/IO theo container
# cAdvisor (tùy chọn):
docker run -d --name cadvisor -p 8081:8080 \
  -v /:/rootfs:ro -v /var/run:/var/run:ro -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro gcr.io/cadvisor/cadvisor:latest
# mở http://localhost:8081
```

---

## 6. Chẩn đoán có hệ thống — 3 tình huống dựng sẵn

**A. CrashLoop:** chạy với lệnh sai
```bash
docker run -d --name crash --restart on-failure smartapp:1.0 python /no/such.py
docker logs crash ; docker inspect crash -f 'exit={{.State.ExitCode}}'   # exit=2, đọc log để biết nguyên nhân
```
**B. Mạng:** app không phản hồi
```bash
docker run -d --name nohealth smartapp:1.0
# image slim KHÔNG có curl/wget -> test bằng python (có sẵn trong image):
docker exec nohealth python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8080/health').read())" \
  || echo "kiểm tra port/route/process"
# hoặc test từ HOST (không phụ thuộc tool trong container):
IP=$(docker inspect -f '{{.NetworkSettings.IPAddress}}' nohealth); curl -s "$IP:8080/health"
```
**C. OOM:** như mục 4 → sửa `--memory`

**D. Xung đột port:**
```bash
docker run -d --name p1 -p 8080:8080 smartapp:1.0
docker run -d --name p2 -p 8080:8080 smartapp:1.0 || echo ">> port 8080 đã bị chiếm"
```
**E. Ghi vào read-only filesystem:**
```bash
docker run --rm --read-only smartapp:1.0 sh -c 'echo x > /data.txt' 2>&1 || echo ">> read-only chặn ghi; cần --tmpfs hoặc volume"
```
**F. Thiếu biến môi trường:**
```bash
docker run --rm smartapp:1.0 sh -c 'echo "API_KEY=$API_KEY"; test -n "$API_KEY" || echo ">> thiếu env"'
```

Áp dụng quy trình cho cả 6 tình huống: **Triệu chứng → Thu thập (logs/inspect/metrics) → Khoanh vùng → Nguyên nhân → Khắc phục & xác minh.**

---

## 7. (Nâng cao) Dựng stack quan sát bằng Docker Compose

Cả 5 thành phần (cAdvisor + Prometheus + Grafana + Loki + Promtail) lên cùng một lúc, chung mạng nên gọi nhau theo tên service.

`prometheus.yml` (đã tạo ở mục 0; nếu chưa, tạo lại):
```yaml
global: { scrape_interval: 15s }
scrape_configs:
  - job_name: cadvisor
    static_configs: [ { targets: ['cadvisor:8080'] } ]
```

`promtail.yml`:
```yaml
server: { http_listen_port: 9080 }
positions: { filename: /tmp/positions.yaml }
clients: [ { url: http://loki:3100/loki/api/v1/push } ]
scrape_configs:
  - job_name: docker
    static_configs:
      - targets: [localhost]
        labels: { job: docker, __path__: /var/log/containers/*/*.log }
```

`docker-compose.yml`:
```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports: ["8081:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes: [ "./prometheus.yml:/etc/prometheus/prometheus.yml:ro" ]
    restart: unless-stopped
  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true        # khỏi đăng nhập cho lab
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro   # datasource provisioned sẵn
    restart: unless-stopped
  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]
    restart: unless-stopped
  promtail:
    image: grafana/promtail:latest
    command: -config.file=/etc/promtail/config.yml
    volumes:
      - /var/lib/docker/containers:/var/log/containers:ro
      - ./promtail.yml:/etc/promtail/config.yml:ro
    restart: unless-stopped
```
`ds.yml` (data source tự nạp khi Grafana khởi động — KHỎI cấu hình tay):
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```
```bash
mkdir -p grafana/provisioning/datasources && mv ds.yml grafana/provisioning/datasources/ 2>/dev/null
docker compose up -d && docker compose ps
```

> Mở http://<ip>:3000 → **Explore**:

**Metrics (Prometheus / Explore → Prometheus):**
```promql
rate(container_cpu_usage_seconds_total{name="s"}[1m])     # CPU của container 'name'
container_memory_usage_bytes{name="s"}                    # RAM — theo dõi rò rỉ / tiệm cận limit
container_network_receive_bytes_total{name="s"}           # lưu lượng mạng
```
**Logs (Loki / Explore → Loki):**
```logql
{job="docker"} |= "smartapp"        # gom log smartapp
{job="docker"} |~ "(?i)error|traceback|oom"   # lọc lỗi
```
(Muốn có dashboard sẵn: Grafana → Import → ID `14282` — dashboard cAdvisor cộng đồng.)

- Dừng/dọn: `docker compose down`.

**Mục tiêu:** biết tra **đúng chỉ số/log cần xem** khi sự cố (CPU/RAM tăng bất thường, lỗi trong log) — không phải mất thời gian dựng monitoring.

---

## Break it / Fix it
1. App "chết" không rõ lý do → `docker inspect` thấy `OOMKilled=true`, `exit=137`. **Fix:** tăng memory / sửa rò rỉ.
2. Distroless không debug được bằng exec → **Fix:** ephemeral debug container (mục 2).

## Dọn dẹp
```bash
docker rm -f s sh oom crash nohealth p1 p2 2>/dev/null
docker compose down 2>/dev/null        # hạ stack quan sát (mục 7)
```
