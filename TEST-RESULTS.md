# Kết quả kiểm thử — MyMiniCloud

Thời điểm chạy: 29/10/2026
Môi trường: Docker Desktop 29.1.5 (Windows).
(Lần chạy thử máy còn Apache/MySQL chiếm cổng 80/3306 nên proxy và DB được ánh xạ
tạm sang 8088/3307; kết quả tương đương với cổng 80/3306 của đề.)

## Trạng thái container

```
api-gateway-proxy-server              Up
application-backend-server            Up
authentication-identity-server        Up
internal-dns-server                   Up
monitoring-grafana-dashboard-server   Up
monitoring-logging-server             Up
monitoring-node-exporter-server       Up
monitoring-prometheus-server          Up
object-storage-server                 Up
relational-database-server            Up
web-frontend-server                   Up
```

## 1. Web Frontend Server (Nginx)

| Lệnh | Kết quả |
|---|---|
| `curl -I http://localhost:8080/` | `200` |
| `curl -I http://localhost:8080/blog/` | `200` |
| nội dung trang chủ | `<h1>MyMiniCloud &ndash; Home</h1>` → hiển thị **MyMiniCloud – Home** |
| nội dung trang blog | `<h1>MyMiniCloud Blog</h1>` |

## 2. Application Backend Server (Flask)

| Lệnh | Kết quả |
|---|---|
| `curl http://localhost:8085/hello` | `{"message":"Hello from App Server!"}` |
| `curl http://localhost/api/hello` | `{"message":"Hello from App Server!"}` (qua proxy, đã strip `/api/`) |

## 3. Relational Database Server (MariaDB)

```
$ docker exec relational-database-server mariadb -uroot -proot \
    -e "USE minicloud; SHOW TABLES; SELECT * FROM notes;"

Tables_in_minicloud
notes
id  title                   created_at
1   Hello from MariaDB!     2026-10-29 06:37:28
```

## 4. Authentication Identity Server (Keycloak)

| Lệnh | Kết quả |
|---|---|
| `curl -I http://localhost:8081/` | `302` (redirect trang đăng nhập) |
| `curl http://localhost:8081/realms/master` | `200` |
| Đăng nhập UI | admin / admin |

## 5. Object Storage Server (MinIO)

```
$ mc alias set local http://127.0.0.1:9000 minioadmin minioadmin
$ mc mb local/demo
$ mc cp /tmp/index.html local/demo/index.html

$ mc ls local
[2026-10-29 06:39:09 UTC]     0B demo/

$ mc ls local/demo
[2026-10-29 06:39:09 UTC]    27B STANDARD index.html
```

Console: http://localhost:9001 → `200`, đăng nhập minioadmin / minioadmin.

## 6. Internal DNS Server (Bind9)

```
$ docker exec internal-dns-server dig +short @127.0.0.1 web-frontend-server.cloud.local
10.10.10.10
```

## 7. Monitoring (Prometheus + Node Exporter)

```
$ curl -s "http://localhost:9090/api/v1/query?query=up"
job=node up=1
```

`http://localhost:9090/targets` → `monitoring-node-exporter-server:9100` **UP**.

## 8. Logging / Visualization (Grafana + Loki)

| Kiểm tra | Kết quả |
|---|---|
| `curl -I http://localhost:3000/login` | `200` |
| `curl http://localhost:3100/ready` | `ready` |
| Datasource tự nạp | `Prometheus → http://monitoring-prometheus-server:9090` |
| | `Loki → http://monitoring-logging-server:3100` |
| Promtail đẩy log | `GET /loki/api/v1/label/job/values` → `{"status":"success","data":["docker"]}` |
| Truy vấn log | `{job="docker"}` trả về stream log container |

## 9. API Gateway / Reverse Proxy (Nginx)

| Lệnh | Kết quả |
|---|---|
| `curl -I http://localhost/` | `200` (web) |
| `curl -s http://localhost/api/hello` | `{"message":"Hello from App Server!"}` |
| `curl -I http://localhost/auth/` | `302 Found` (Keycloak login) |

## Kiểm tra thông mạng giữa các container

Ping từ container trung gian trong `cloud-net` tới 9 server: **OK 9/9**.

Phân giải alias nội bộ (dùng đúng như lệnh trong đề):

| Alias | IP |
|---|---|
| web | 172.19.0.4 |
| app-server | 172.19.0.6 |
| db | 172.19.0.12 |
| keycloak | 172.19.0.7 |
| minio | 172.19.0.9 |
| dns-server | 172.19.0.10 |
| prometheus | 172.19.0.8 |
| grafana | 172.19.0.3 |
| loki | 172.19.0.2 |

## Ghi chú khi chụp ảnh minh chứng

Trước khi chạy phải bảo đảm cổng 80 và 3306 còn trống (tắt Apache/MySQL nếu có),
khi đó URL proxy đúng như đề: `http://localhost/`.
Khi deploy EC2 (không bận cổng), dùng `docker compose up -d` và URL là
`http://<public-ip>/` đúng như đề.
