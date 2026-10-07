# Báo Cáo Thực Hành: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## Thông tin bài tập
- **Học viên:** Nguyễn Quang Vinh (Vinh Nguyen)
- **Mã lớp:** CNTT1 (DevOps IT209)
- **Môn học:** DevOps (IT209) - Session 07
- **Bài tập:** Bài 4 (EX4) - Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot
- **Đường dẫn nộp bài trên GitHub:** `homework/session_07/ex4/`
- **Kho lưu trữ GitHub:** [QuangVinh1605/IT209-SS07-CAU5](https://github.com/QuangVinh1605/IT209-SS07-CAU5)

---

## 1. Mục tiêu bài thực hành
1. **Thiết lập máy chủ ảo Nginx (Server Block) làm Reverse Proxy:** Cấu hình Nginx tiếp nhận mọi yêu cầu truy cập tại cổng mặc định 80 và định tuyến lưu lượng thông minh.
2. **Kỹ thuật Path Matching (Khớp đường dẫn URL):** Phân tách rõ ràng giữa lưu lượng tĩnh (Static Web Content) và lưu lượng động (Dynamic API Backend) dựa trên URI prefix:
   - Đường dẫn gốc `/`: Nginx phục vụ trực tiếp file tĩnh HTML/CSS/JS từ Document Root mà không cần gọi đến Backend application server, giúp tối ưu hiệu năng và giải phóng tải CPU.
   - Tiền tố `/api/`: Nginx đóng vai trò Reverse Proxy chuyển tiếp (`proxy_pass`) an toàn tới dịch vụ Spring Boot đang chạy ngầm trên cổng `8082`.
3. **Bảo toàn thông tin Client qua HTTP Reverse Proxy Headers:** Thiết lập các header quan trọng (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) để Backend nhận biết chính xác IP thật và giao thức của người dùng cuối.

---

## 2. Kiến trúc & Sơ đồ luồng dữ liệu (Request Flow)

```text
                           +----------------------------------------+
                           |           Client / Browser             |
                           +----------------------------------------+
                                        |             |
                         GET /          |             | GET /api/health
                                        v             v
                           +----------------------------------------+
                           |           Nginx Web Server             |
                           |               (Port 80)                |
                           +----------------------------------------+
                                  /                         \
                 (Path Matching: /)                         (Path Matching: /api/)
                                /                             \
                               v                               v (proxy_pass)
              +-------------------------------+  +-------------------------------+
              |         Document Root         |  |      Spring Boot Backend      |
              |       /var/www/html/          |  |       (127.0.0.1:8082)        |
              |        (index.html)           |  |      (spring-app.service)     |
              +-------------------------------+  +-------------------------------+
```

---

## 3. Quy trình thực hiện & Cấu hình chi tiết

### 3.1. Bước 1: Khởi tạo trang tĩnh chứa thông tin học viên tại `/var/www/html/index.html`
Tạo tệp giao diện HTML tĩnh hiển thị đầy đủ Họ tên, Mã lớp tại thư mục gốc Document Root:

```bash
sudo tee /var/www/html/index.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Thông tin học viên - DevOps IT209</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background: linear-gradient(135deg, #eef2f3 0%, #8e9eab 100%);
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 90vh;
        }
        .container {
            background-color: #ffffff;
            border-radius: 12px;
            box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
            padding: 36px 44px;
            max-width: 520px;
            width: 100%;
            text-align: center;
        }
        h1 { color: #2c3e50; font-size: 24px; margin-bottom: 6px; }
        .subtitle { color: #7f8c8d; font-size: 14px; margin-bottom: 24px; }
        .info-card {
            background: #f8f9fa;
            border-left: 4px solid #007bff;
            border-radius: 6px;
            padding: 12px 18px;
            margin-bottom: 14px;
            text-align: left;
        }
        .label { font-weight: 600; color: #495057; font-size: 13px; text-transform: uppercase; }
        .value { color: #212529; font-size: 17px; font-weight: 500; margin-top: 4px; }
        .tag {
            display: inline-block;
            background-color: #28a745;
            color: #ffffff;
            font-size: 13px;
            font-weight: 600;
            padding: 6px 16px;
            border-radius: 20px;
            margin-top: 18px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>DevOps IT209 - Session 07</h1>
        <div class="subtitle">Bài 4: Reverse Proxy Nginx cho ứng dụng Spring Boot</div>
        
        <div class="info-card">
            <div class="label">Họ và tên học viên:</div>
            <div class="value">Nguyễn Quang Vinh (Vinh Nguyen)</div>
        </div>

        <div class="info-card">
            <div class="label">Mã lớp:</div>
            <div class="value">CNTT1 (DevOps IT209)</div>
        </div>

        <div class="info-card">
            <div class="label">Cấu hình định tuyến (Nginx Routing):</div>
            <div class="value">/ &rarr; Static Serve (/var/www/html)<br>/api/ &rarr; Reverse Proxy (http://127.0.0.1:8082/)</div>
        </div>

        <div class="tag">&check; Nginx Reverse Proxy Active</div>
    </div>
</body>
</html>
EOF
sudo chmod 644 /var/www/html/index.html
```

---

### 3.2. Bước 2: Tạo tệp cấu hình Nginx Server Block (`spring-proxy.conf`)
Tạo tệp cấu hình tại `/etc/nginx/sites-available/spring-proxy.conf`:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    root /var/www/html;
    index index.html index.htm;

    # 1. Phục vụ trang tĩnh tại /var/www/html/
    location / {
        try_files $uri $uri/ =404;
    }

    # 2. Chuyển tiếp các yêu cầu API đến Spring Boot backend (port 8082)
    location /api/ {
        proxy_pass http://127.0.0.1:8082/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Phân tích chi tiết các chỉ thị:
- **`listen 80 default_server`**: Nginx lắng nghe tại cổng HTTP 80 tiêu chuẩn và đóng vai trò server block mặc định cho mọi request không khớp domain cụ thể.
- **`location /`**: Khối định tuyến phục vụ tài nguyên tĩnh. Chỉ thị `try_files $uri $uri/ =404` sẽ kiểm tra file hoặc thư mục tương ứng trong `/var/www/html`, nếu không tồn tại sẽ trả về lỗi HTTP 404 từ chính Nginx mà không làm phiền đến backend.
- **`location /api/`**: Khớp mọi request có URI bắt đầu bằng `/api/` để ủy quyền xử lý cho Spring Boot.
- **`proxy_pass http://127.0.0.1:8082/`**: Địa chỉ upstream của backend application server.
- **`proxy_set_header Host $host`**: Giữ nguyên header `Host` ban đầu từ Client, giúp backend biết được domain/host gốc mà người dùng gửi tới.
- **`proxy_set_header X-Real-IP $remote_addr`**: Gửi địa chỉ IP thực sự của Client đến backend thay vì IP của Reverse Proxy (`127.0.0.1`), quan trọng đối với việc ghi log và bảo mật (rate limit, whitelist/blacklist).
- **`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for`**: Danh sách chuỗi các proxy mà client đã đi qua.
- **`proxy_set_header X-Forwarded-Proto $scheme`**: Gửi giao thức gốc (http/https) để backend có thể sinh các redirect URL chính xác.

---

### 3.3. Bước 3: Kích hoạt cấu hình (Enable Site) qua Symlink
Tạo liên kết mềm từ `sites-available` sang `sites-enabled`:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sf /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
```

Kiểm tra danh sách symlink trong `sites-enabled`:
```bash
ls -la /etc/nginx/sites-enabled/
```
*Kết quả:*
```text
lrwxrwxrwx 1 root root   44 Oct  8 00:37 spring-proxy.conf -> /etc/nginx/sites-available/spring-proxy.conf
```

---

### 3.4. Bước 4: Kiểm tra cú pháp cấu hình (`nginx -t`)
Luôn kiểm tra tính hợp lệ của cú pháp cấu hình trước khi reload:

```bash
sudo nginx -t
```

*Kết quả đầu ra thực tế:*
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### 3.5. Bước 5: Nạp lại dịch vụ Nginx (Zero Downtime Reload)
Nạp lại cấu hình mà không làm gián đoạn các kết nối đang hoạt động:

```bash
sudo systemctl reload nginx
sudo systemctl status nginx --no-pager
```

*Kết quả:* Nginx ở trạng thái `active (running)`.

---

## 4. Nhật ký kiểm tra & Xác thực kết quả (Verification)

### 4.1. Kiểm tra 1: Cú pháp Nginx
```bash
sudo nginx -t
```
*Kết quả đầu ra:*
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
> **Đánh giá:** Đạt yêu cầu — cú pháp chính xác, kiểm tra thành công.

---

### 4.2. Kiểm tra 2: Truy vấn HTTP Header trang chủ tĩnh (`curl -I http://localhost/`)
```bash
curl -I http://localhost/
```
*Kết quả đầu ra:*
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 07 Oct 2026 17:38:41 GMT
Content-Type: text/html
Content-Length: 2845
Last-Modified: Wed, 07 Oct 2026 17:37:16 GMT
Connection: keep-alive
ETag: "6ac6834c-b1d"
Accept-Ranges: bytes
```
> **Đánh giá:** Trả về mã trạng thái **`HTTP 200 OK`**, phục vụ trực tiếp từ máy chủ Nginx.

---

### 4.3. Kiểm tra 3: Truy vấn nội dung trang chủ tĩnh chứa thông tin học viên (`curl http://localhost/`)
```bash
curl -s http://localhost/
```
*Trích đoạn nội dung hiển thị thực tế:*
```html
<div class="container">
    <h1>DevOps IT209 - Session 07</h1>
    <div class="subtitle">Bài 4: Reverse Proxy Nginx cho ứng dụng Spring Boot</div>
    
    <div class="info-card">
        <div class="label">Họ và tên học viên:</div>
        <div class="value">Nguyễn Quang Vinh (Vinh Nguyen)</div>
    </div>

    <div class="info-card">
        <div class="label">Mã lớp:</div>
        <div class="value">CNTT1 (DevOps IT209)</div>
    </div>

    <div class="info-card">
        <div class="label">Cấu hình định tuyến (Nginx Routing):</div>
        <div class="value">/ &rarr; Static Serve (/var/www/html)<br>/api/ &rarr; Reverse Proxy (http://127.0.0.1:8082/)</div>
    </div>

    <div class="tag">&check; Nginx Reverse Proxy Active</div>
</div>
```
> **Đánh giá:** Trang chủ hiển thị chuẩn xác thông tin học viên: **Nguyễn Quang Vinh**, Mã lớp: **CNTT1 (DevOps IT209)**.

---

### 4.4. Kiểm tra 4: Kiểm tra Reverse Proxy API (`curl -I http://localhost/api/health`)
```bash
curl -I http://localhost/api/health
```
*Kết quả đầu ra:*
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 07 Oct 2026 17:38:53 GMT
Content-Type: application/json
Connection: keep-alive
```

---

### 4.5. Kiểm tra 5: Truy vấn dữ liệu API từ Spring Boot backend (`curl -i http://localhost/api/health`)
```bash
curl -i http://localhost/api/health
```
*Kết quả đầu ra:*
```text
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Wed, 07 Oct 2026 17:38:54 GMT
Content-Type: application/json
Content-Length: 73
Connection: keep-alive

{"status":"UP","message":"Spring Boot Application running on port 8082"}
```
> **Đánh giá:** 
> - Nginx tiếp nhận request trên cổng `80` tại `/api/health`, sau đó forward thành công tới Spring Boot tại `http://127.0.0.1:8082`.
> - Trả về mã **`HTTP 200 OK`** kèm payload JSON từ ứng dụng Spring Boot backend.
> - Hoàn toàn không bị lỗi `502 Bad Gateway`.

---

## 5. Phân tích chuyên sâu (Deep Dive)

### 5.1. So sánh Forward Proxy và Reverse Proxy

| Tiêu chí | Forward Proxy | Reverse Proxy |
| :--- | :--- | :--- |
| **Vị trí đứng** | Đứng trước Client (đại diện cho Client) | Đứng trước Server (đại diện cho Server) |
| **Mục đích chính** | Ẩn danh Client, vượt tường lửa, kiểm soát truy cập ra ngoài Internet | Cân bằng tải (Load Balancing), SSL Offloading, Web Acceleration, Cache |
| **Khả năng nhận biết** | Client chủ động cấu hình proxy để gửi request | Client hoàn toàn không biết có Reverse Proxy đứng chắn trước |
| **Ví dụ thực tế** | Proxy công ty chặn mạng xã hội, VPN | Nginx, HAProxy, AWS ALB đứng trước Java/NodeJS |

---

### 5.2. Tầm quan trọng của Reverse Proxy Headers trong môi trường Production
Nếu không có các chỉ thị `proxy_set_header`:
1. **Lỗi nhận diện IP người dùng (`X-Real-IP`):** Ứng dụng Backend Spring Boot sẽ nhận thấy mọi request đều đến từ IP `127.0.0.1` (loopback của máy chủ). Hậu quả:
   - Các tính năng như Giới hạn tần suất gọi API (Rate Limiting) sẽ chặn nhầm toàn bộ người dùng.
   - Nhật ký kiểm toán bảo mật (Audit log) không thể truy vết được IP thực của kẻ tấn công.
2. **Lỗi điều hướng và SSL (`Host`, `X-Forwarded-Proto`):** Khi Spring Boot sinh đường dẫn OAuth redirect hoặc phân trang (HATEOAS), nếu thiếu `Host` và `Proto`, nó sẽ sinh ra URL nội bộ dạng `http://127.0.0.1:8082/...` thay vì domain chính của website, gây lỗi vỡ luồng người dùng.

---

## 6. Tổng kết danh mục tệp nộp bài

- [spring-proxy.conf](file:///home/vinh/Desktop/devoops/ss07/homework/session_07/ex4/spring-proxy.conf): File cấu hình Server Block Nginx Reverse Proxy.
- [README.md](file:///home/vinh/Desktop/devoops/ss07/homework/session_07/ex4/README.md): Báo cáo chi tiết quá trình cấu hình, kết quả kiểm tra `nginx -t` và `curl`.
