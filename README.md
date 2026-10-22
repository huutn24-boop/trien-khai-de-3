# WordPress DevOps Assignment

Stack triển khai WordPress, MySQL, phpMyAdmin, Nginx HTTPS, Prometheus, cAdvisor, Grafana, Loki và Promtail bằng Docker Compose.

> Đây là cấu hình phục vụ học tập/thực hành. Chứng chỉ tự ký, các image được ghim theo phiên bản của bài tập và một số dịch vụ cần quyền đọc thông tin Docker host; không dùng nguyên trạng cho môi trường production.

## Kiến trúc

```text
Browser -- HTTP :80 / HTTPS :443 --> Nginx
                                      |--> WordPress (Apache) --> MySQL
                                      `--> phpMyAdmin ----------> MySQL

Prometheus --> cAdvisor
Grafana --> Prometheus + Loki
Promtail --> Docker Socket Proxy (chỉ API đọc) + Docker container logs --> Loki
```

| Network | Thành viên | Mục đích |
| --- | --- | --- |
| `frontend-net` | Nginx, WordPress, phpMyAdmin | Lưu lượng ứng dụng; chỉ Nginx mở cổng web ra host |
| `backend-net` | WordPress, phpMyAdmin, MySQL | Database; mạng `internal`, không publish cổng MySQL |
| `monitoring-net` | Prometheus, Grafana, Loki, Promtail, cAdvisor, Docker Socket Proxy | Metrics và logs; mạng `internal` |

WordPress dùng image Apache vì Nginx proxy HTTP tới `wordpress:80`. Image FPM (`*-fpm`) chỉ cung cấp FastCGI và không thể dùng với cấu hình `proxy_pass http://wordpress:80`.

## Yêu cầu

- Docker Engine với Docker Compose v2 (Docker Desktop trên Windows/macOS hoặc Docker Engine trên Linux).
- OpenSSL có trong `PATH`.
- Tối thiểu khoảng 3 GB RAM còn trống được khuyến nghị cho toàn bộ stack.

## Cấu hình và chạy

1. Tạo `.env` từ mẫu rồi đặt mật khẩu riêng, dài và ngẫu nhiên cho ba giá trị cần thiết. Không commit `.env`.

   PowerShell:

   ```powershell
   Copy-Item .env.example .env
   notepad .env
   ```

   Tạo giá trị ngẫu nhiên bằng `openssl rand -hex 32`. Điền lần lượt vào `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, `GRAFANA_ADMIN_PASSWORD`. Không dùng lại mật khẩu và không để trống.

2. Tạo chứng chỉ HTTPS tự ký dùng riêng cho máy local:

   ```powershell
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx\ssl\nginx.key -out nginx\ssl\nginx.crt -subj "/C=VN/ST=Hanoi/L=Hanoi/O=DevOps/OU=IT/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
   ```

   Hai file chứng chỉ/khóa được loại khỏi Git. Trình duyệt sẽ cảnh báo vì chứng chỉ tự ký; không sử dụng khóa này cho website công khai.

3. Kiểm tra cấu hình và khởi chạy:

   ```powershell
   docker compose --env-file .env config --quiet
   docker compose up -d
   docker compose ps
   ```

4. Mở:

   - WordPress: <https://localhost>
   - phpMyAdmin: <https://localhost/phpmyadmin/>
   - Grafana: <http://127.0.0.1:3000> (user mặc định `admin`, mật khẩu đặt trong `.env`)

   Đăng nhập phpMyAdmin bằng `wp_user` và `MYSQL_PASSWORD`. Không dùng tài khoản root trong trình duyệt.

Tắt stack bằng `docker compose down`. Dữ liệu được lưu trong named volumes; `docker compose down -v` sẽ xóa cả database, nội dung WordPress, dashboard và dữ liệu metrics/logs.

## Monitoring và LogQL

Datasource Prometheus và Loki được provision tự động trong Grafana. Trong Grafana, mở **Explore** rồi chọn **Loki**:

1. Toàn bộ log Nginx:

   ```logql
   {container_name="wp_nginx"}
   ```

2. Dòng log có chứa `error` từ các container trong stack:

   ```logql
   {container_name=~"wp_.*"} |= "error"
   ```

3. Tốc độ log Nginx theo thời gian:

   ```logql
   rate({container_name="wp_nginx"}[1m])
   ```

Các truy vấn PromQL hữu ích trong datasource **Prometheus**:

```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m])
container_memory_working_set_bytes{name!=""}
rate(container_network_receive_bytes_total{name!=""}[5m])
rate(container_network_transmit_bytes_total{name!=""}[5m])
```

Prometheus scrape cAdvisor tại `cadvisor:8080`. Trên Docker Desktop, cAdvisor quan sát Linux VM của Docker Desktop, không phải toàn bộ host Windows/macOS.

Promtail lấy tên container và đường dẫn log qua Docker discovery. Docker socket được đặt sau `docker-socket-proxy`; proxy chỉ bật các endpoint đọc cần thiết (`POST=0`) và chỉ tham gia monitoring network. Tuy vậy, quyền truy cập Docker API vẫn nhạy cảm; không publish cổng proxy ra host hoặc Internet. Việc đọc `/var/lib/docker/containers` phụ thuộc Docker Engine và có thể cần điều chỉnh trên các bản Docker Desktop khác nhau.

## Hardening và giới hạn

- MySQL không publish port; `backend-net` và `monitoring-net` là các mạng nội bộ tách biệt.
- Prometheus (UID 65534), Grafana (UID 472) và Loki (UID 10001) chạy với UID không-root. Promtail cần quyền root trong container để đọc log Docker trên Linux. MySQL, WordPress và Nginx dùng entrypoint mặc định của image, do đó không được xem là non-root.
- cAdvisor không chạy `privileged` và chỉ mount các đường dẫn host ở chế độ read-only; vẫn thu thập thông tin hệ thống/container nhạy cảm nên cần giới hạn người có quyền chạy stack.
- Grafana chỉ bind vào loopback host. Dùng SSH tunnel hoặc cấu hình reverse proxy có xác thực nếu cần truy cập từ xa.
- Compose đặt giới hạn CPU/RAM cho các dịch vụ. Có thể điều chỉnh các giá trị `deploy.resources.limits` theo tài nguyên máy.
- Nginx bật HTTPS, chuyển hướng HTTP sang HTTPS và thêm security headers. CSP hiện cần cho ứng dụng WordPress phổ thông; có thể cần tinh chỉnh nếu plugin sử dụng nguồn nội dung bên ngoài.
- Các phiên bản image trong Compose bám theo đề bài thực hành và có thể cũ. Trước khi triển khai công khai, hãy chuyển sang phiên bản còn được hỗ trợ, cập nhật thường xuyên, kiểm tra tương thích và dùng chứng chỉ từ CA tin cậy.

## Nội dung báo cáo thực hành (tối thiểu 10 trang)

- **Bìa:** logo trường, tên môn học, đề tài, họ tên/MSSV/lớp và giảng viên hướng dẫn.
- **Mục lục, danh mục hình và bảng.**
- **Chương 1 – Giới thiệu tổng quan (2–3 trang):** bài toán website WordPress; kiến trúc Compose; vai trò WordPress, MySQL, Nginx, Prometheus, Grafana, Loki và Promtail.
- **Chương 2 – Cấu trúc và cách hoạt động (2 trang):** sơ đồ network; request từ trình duyệt qua Nginx tới WordPress và MySQL; luồng metrics qua cAdvisor/Prometheus; luồng logs qua Promtail/Loki.
- **Chương 3 – Kết quả triển khai và demo (4–5 trang):** ảnh lịch sử Git/README; WordPress và phpMyAdmin; HTTPS và response headers; dashboard metrics; ba truy vấn LogQL; compose minh họa network isolation, non-root users và resource limits.
- **Chương 4 – Kết luận và hướng phát triển (1 trang):** đánh giá kết quả, bài học về Compose/reverse proxy/monitoring/logging và các hướng cải tiến.