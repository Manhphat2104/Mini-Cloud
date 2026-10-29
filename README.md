# MyMiniCloud — Mô phỏng Cloud Platform cơ bản

Đồ án cuối kỳ: dựng 9 loại server của một Cloud Platform (giống AWS/Azure/GCP)
bằng Docker, tất cả chạy trong cùng network `cloud-net`, triển khai bằng
`docker-compose.yml`.

## Cấu trúc thư mục

```
huynhmanhphatminiclouddemo/
├── docker-compose.yml                    # 9 server + node-exporter + loki
├── README.md
├── TEST-RESULTS.md                       # log kiểm thử thực tế (minh chứng)
├── web-frontend-server/                  # 1. Web Server (Nginx static)
│   ├── Dockerfile
│   ├── conf.default
│   └── html/
│       ├── index.html
│       └── blog/index.html
├── application-backend-server/           # 2. Application Server (Flask API)
│   ├── Dockerfile
│   └── app.py                            # /hello, /secure
├── relational-database-server/           # 3. Database Server (MariaDB)
│   ├── init/001_init.sql                 # db minicloud + bảng notes
│   └── data/                             # volume dữ liệu (không commit)
├── authentication-identity-server/       # 4. Auth Server (Keycloak)
├── object-storage-server/                # 5. Object Storage (MinIO + Console)
│   └── data/                             # volume dữ liệu (không commit)
├── internal-dns-server/                  # 6. DNS Server (Bind9)
│   ├── named.conf                        # include 2 file dưới
│   ├── named.conf.options
│   ├── named.conf.local
│   └── db.cloud.local                    # zone cloud.local
├── monitoring-prometheus-server/         # 7. Monitoring (Prometheus)
│   └── prometheus.yml
├── monitoring-grafana-dashboard-server/  # 8. Logging / Visualization (Grafana + Loki)
│   ├── loki-config.yml
│   ├── promtail-config.yml
│   └── provisioning/datasources/datasources.yml
└── api-gateway-proxy-server/             # 9. Reverse Proxy (Nginx)
    └── nginx.conf
```

Node Exporter dùng image chính thức, không cần thư mục cấu hình riêng.

## Cổng

| Server | Cổng host | Ghi chú |
|---|---|---|
| Web | 8080 | |
| App (listen 8081 nội bộ) | 8085 | |
| Database | 3306 | phải trống trước khi chạy |
| Auth | 8081 | |
| MinIO API / Console | 9000 / 9001 | |
| DNS | 1053/udp + 1053/tcp | |
| Prometheus | 9090 | |
| Node Exporter | 9100 | |
| Grafana | 3000 | admin / admin |
| Loki | 3100 | datasource tự cấu hình |
| Proxy | 80 | phải trống trước khi chạy |

## Chạy dự án

Trên EC2 / máy trống cổng 80 và 3306 (đúng như đề):

```bash
docker compose build --no-cache
docker compose up -d
docker compose ps
```

Promtail (đẩy log container vào Loki) chạy mặc định cùng hệ thống; nếu môi trường
không cho mount `/var/lib/docker/containers` thì tạm bỏ service `monitoring-promtail`
khỏi `docker-compose.yml`.

## Lệnh kiểm thử

```bash
curl -I http://localhost:8080/                 # 200 OK
curl -I http://localhost:8080/blog/            # 200 OK
curl http://localhost:8085/hello               # {"message":"Hello from App Server!"}
curl http://localhost/api/hello                # app qua proxy
docker exec relational-database-server mariadb -uroot -proot \
  -e "USE minicloud; SHOW TABLES; SELECT * FROM notes;"
docker exec internal-dns-server dig +short @127.0.0.1 web-frontend-server.cloud.local   # 10.10.10.10
curl -s http://localhost:9090/api/v1/targets   # node = up
curl -I http://localhost/auth/                 # 302 -> Keycloak
```

### Gọi API bảo vệ `/secure` (OIDC)

```bash
# 1) lấy access token từ Keycloak
TOKEN=$(curl -s -X POST http://localhost:8081/realms/master/protocol/openid-connect/token \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=admin" \
  | jq -r .access_token)

# 2) không có token -> 401
curl -s http://localhost/api/secure
# {"error":"Missing Bearer token"}

# 3) có token -> 200
curl -s -H "Authorization: Bearer $TOKEN" http://localhost/api/secure
# {"message":"Secure resource OK","preferred_username":null}
```

PowerShell: `powershell -ExecutionPolicy Bypass -File .\scripts\demo-secure.ps1`

> `preferred_username` là `null` vì Keycloak 24+ dùng *lightweight access token*
> (thông tin user không nằm trong access token mà lấy qua endpoint `/userinfo`).
> Muốn claim này xuất hiện trong token thì thêm protocol mapper "User Property"
> cho client, hoặc tạo client `myapp` + user như phần mở rộng mục 4.

Kết quả chạy thực tế: xem `TEST-RESULTS.md`.

## Khác biệt so với file Word (có lý do)

1. **MinIO**: `minio/minio` đã bị xoá khỏi Docker Hub và binary cũng bị gỡ khỏi
   `dl.min.io` trong 2025; các bản mới còn **bỏ luôn MinIO Console**. Vì vậy dùng
   `bitnamilegacy/minio:2025.3.12` (cùng là MinIO, còn Console) và mount dữ liệu
   vào `/data` như đề.
2. **Bind9**: `named.conf` mặc định của image không include `named.conf.local`
   nên zone `cloud.local` không được nạp. Đã thêm `internal-dns-server/named.conf`
   để include 2 file cấu hình.
3. **`restart: unless-stopped`**: đề yêu cầu cho *mỗi* service, file mẫu chỉ có ở
   2 service → đã bổ sung cho tất cả.
4. **Volume dữ liệu MariaDB**: thêm `./relational-database-server/data:/var/lib/mysql`
   để dữ liệu không mất khi xoá container (đề yêu cầu "volume nếu cần lưu dữ liệu").
5. **Loki + provisioning Grafana**: mục 8 yêu cầu "lưu & hiển thị log", file mẫu
   chỉ có Grafana → thêm Loki, datasource Prometheus/Loki được tự động nạp.
6. **Alias mạng** (`web`, `db`, `keycloak`, `minio`, `dns-server`, `prometheus`,
   `grafana`, `app-server`, `loki`): để các lệnh `ping -c 3 web`, `ping -c 3 db`…
   đúng như đề chạy được.
7. **Cổng 80 và 3306 phải trống** khi chạy (máy có Apache/MySQL thì tắt trước).

## Còn phải làm để đủ điểm

- Push image `huynhmanhphat/web:dev`, `huynhmanhphat/app:dev` lên Docker Hub
  (nếu tài khoản Docker Hub của bạn khác `huynhmanhphat` thì sửa lại trong compose).
- Deploy lên AWS EC2 Ubuntu, mở Security Group cho các cổng, test qua public IP.
- Chụp ảnh minh chứng 9 server + viết báo cáo 5 chương.
- Phần **mở rộng** (5 điểm): blog 3 bài, API `/student`, `studentdb`, realm OIDC,
  bucket `profile-pics`, DNS record mới, scrape target `web`, dashboard Grafana,
  route `/student/`, load balancing round-robin.
