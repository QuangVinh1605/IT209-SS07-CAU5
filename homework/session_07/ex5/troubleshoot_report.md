# Báo Cáo Sự Cố: Chẩn Đoán và Xử Lý Xung Đột Cổng Mạng (Address Already in Use)

## Thông tin bài tập
- **Học viên:** Vinh Nguyen
- **Môn học:** DevOps (IT209) - Session 07
- **Bài tập:** Bài 5 (EX5) - Chẩn đoán và xử lý xung đột cổng mạng (Address Already in Use)
- **Đường dẫn nộp bài trên GitHub:** `homework/session_07/ex5/troubleshoot_report.md`
- **Kho lưu trữ GitHub:** [QuangVinh1605/IT209-SS07-CAU5](https://github.com/QuangVinh1605/IT209-SS07-CAU5)

---

## 1. Mục tiêu bài thực hành
1. **Thành thạo công cụ chẩn đoán socket mạng trên Linux:** Sử dụng thành thạo `ss`, `lsof`, `netstat` để truy vấn trạng thái listening sockets, lọc cổng cụ thể, và xác định chính xác PID cùng tiến trình đang chiếm dụng tài nguyên mạng.
2. **Quản lý vòng đời tiến trình hệ thống:** Áp dụng các lệnh `ps`, `kill`, `kill -9` để phân tích thông tin tiến trình và giải phóng tài nguyên một cách an toàn theo đúng thứ tự ưu tiên (Graceful Termination trước Forced Termination).
3. **Khắc phục lỗi triển khai thực tế:** Xử lý triệt để lỗi kinh điển `java.net.BindException: Address already in use` khi khởi chạy dịch vụ Spring Boot (`spring-app.service`), đảm bảo dịch vụ bàn giao và quản lý cổng `8082` thành công.

---

## 2. Bối cảnh & Mô phỏng sự cố (Incident Reproduction)

### 2.1. Bối cảnh thực tế
Khi khởi chạy hoặc khởi động lại ứng dụng Spring Boot (`spring-app.service`) trên máy chủ, dịch vụ thất bại không thể khởi động do cổng `8082` đã bị một tiến trình khác chạy ngầm chiếm dụng trước đó.

### 2.2. Giả lập cổng 8082 bị chiếm dụng
Thực hiện giả lập tiến trình chiếm giữ cổng `8082` bằng lệnh chạy nền:

```bash
python3 -m http.server 8082 > /dev/null 2>&1 &
```

Kiểm tra xác nhận tiến trình giả lập đã lắng nghe trên cổng 8082:
```bash
sudo ss -tlnp | grep 8082
```
*Kết quả ghi nhận:*
```text
LISTEN 0      5            0.0.0.0:8082       0.0.0.0:*    users:(("python3",pid=61055,fd=10))
```

---

### 2.3. Hiện tượng lỗi khi khởi động Spring Boot Service
Khi tiến hành khởi động dịch vụ Spring Boot `spring-app.service`:

```bash
sudo systemctl restart spring-app.service
```

Kiểm tra trạng thái dịch vụ bằng `systemctl status`:
```bash
sudo systemctl status spring-app.service --no-pager
```

*Nhật ký lỗi ghi nhận trên hệ thống:*
```text
● spring-app.service - Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Wed 2026-10-07 21:10:40 +07; 1s ago
 Invocation: 3f128f26fbe64eb282ea08f39c3f3f03
    Process: 61565 ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar (code=exited, status=1/FAILURE)
   Main PID: 61565 (code=exited, status=1/FAILURE)
   Mem peak: 22.1M
        CPU: 131ms
```

Truy vấn nhật ký chi tiết qua `journalctl`:
```bash
sudo journalctl -u spring-app.service -n 25 --no-pager
```

*Chi tiết thông báo lỗi từ Spring Boot:*
```text
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: ***************************
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: APPLICATION FAILED TO START
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: ***************************
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: Description:
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: Web server failed to start. Port 8082 was already in use.
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: Action:
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: Identify and stop the process that's listening on port 8082 or configure this application to listen on another port.
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]: java.net.BindException: Address already in use
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]:         at java.base/sun.nio.ch.Net.bind0(Native Method)
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]:         at java.base/sun.nio.ch.Net.bind(Net.java:567)
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]:         at java.base/sun.nio.ch.ServerSocketChannelImpl.netBind(ServerSocketChannelImpl.java:337)
Thg 10 07 21:10:40 vinh-ThinkPad-E15 java[61565]:         at java.base/sun.nio.ch.ServerSocketChannelImpl.bind(ServerSocketChannelImpl.java:294)
Thg 10 07 21:10:40 vinh-ThinkPad-E15 systemd[1]: spring-app.service: Main process exited, code=exited, status=1/FAILURE
Thg 10 07 21:10:40 vinh-ThinkPad-E15 systemd[1]: spring-app.service: Failed with result 'exit-code'.
```

> **Nguyên nhân cốt lõi (Root Cause):** Hệ điều hành ném lỗi `java.net.BindException: Address already in use` do cổng TCP `8082` đã bị bind và chiếm dụng bởi một tiến trình khác, khiến Web Server nhúng (Tomcat/HttpServer) không thể khởi tạo socket lắng nghe.

---

## 3. Quy trình chẩn đoán & Xác định tiến trình xung đột

### 3.1. Phương pháp 1: Sử dụng công cụ `ss` (Socket Statistics)
`ss` là công cụ chuẩn hiện đại trên Linux kernel (thay thế cho `netstat` cũ) để truy vấn thông tin socket mạng trực tiếp từ kernel iproute2 subsystem.

```bash
sudo ss -tlnp | grep 8082
```

#### Phân tích các tùy chọn (flags):
- **`-t`** (*TCP*): Chỉ hiển thị các socket thuộc giao thức TCP.
- **`-l`** (*Listening*): Chỉ lọc các socket đang ở trạng thái lắng nghe (`LISTEN`).
- **`-n`** (*Numeric*): Hiển thị cổng và địa chỉ dạng số nguyên, không tra cứu DNS hay tên dịch vụ (giúp câu lệnh thực thi tức thì).
- **`-p`** (*Processes*): Hiển thị thông tin định danh tiến trình (`users:(("process_name",pid=...,fd=...))`) sở hữu socket (yêu cầu quyền `sudo` để xem tiến trình của mọi user).

*Kết quả đầu ra thực tế:*
```text
LISTEN 0      5            0.0.0.0:8082       0.0.0.0:*    users:(("python3",pid=61055,fd=10))
```

---

### 3.2. Phương pháp 2: Sử dụng công cụ `lsof` (List Open Files)
Trong triết lý UNIX/Linux "Everything is a file", socket mạng cũng là một File Descriptor. Lệnh `lsof` cho phép truy vết chính xác file/socket đang mở.

```bash
sudo lsof -i :8082
```

#### Phân tích tùy chọn:
- **`-i :8082`**: Lọc tất cả các file mạng (Internet socket) gắn với cổng số `8082`.

*Kết quả đầu ra thực tế:*
```text
COMMAND   PID USER FD   TYPE  DEVICE SIZE/OFF NODE NAME
python3 61055 vinh 10u  IPv4 1700697      0t0  TCP *:8082 (LISTEN)
```

---

### 3.3. Phương pháp 3: Truy vết thông tin chi tiết tiến trình bằng `ps`
Sau khi có được PID là `61055`, sử dụng `ps` để xem toàn bộ thông tin tiến trình:

```bash
ps -fp 61055
```

*Kết quả đầu ra thực tế:*
```text
UID          PID    PPID  C STIME TTY          TIME CMD
vinh       61055   15380  0 21:10 ?        00:00:00 python3 -m http.server 8082
```

---

## 4. Bảng thông tin tiến trình chiếm dụng cổng 8082

Dưới đây là thông tin chi tiết của tiến trình xung đột được phát hiện trên hệ thống:

| Thuộc tính | Giá trị ghi nhận | Ý nghĩa / Giải thích |
| :--- | :--- | :--- |
| **Process ID (PID)** | **`61055`** | Mã định danh tiến trình của hệ điều hành |
| **Command / Tên tiến trình** | `python3 -m http.server 8082` | Module HTTP Server tích hợp sẵn của Python 3 |
| **Chủ sở hữu (User)** | `vinh` (UID: 1000) | Tài khoản người dùng khởi chạy tiến trình |
| **Cổng mạng chiếm dụng** | `8082` | Cổng TCP phục vụ ứng dụng Spring Boot |
| **Trạng thái cổng (State)** | `LISTEN` | Đang lắng nghe kết nối đến |
| **Địa chỉ ràng buộc (Binding)** | `0.0.0.0:8082` (IPv4) | Lắng nghe trên tất cả các network interface |
| **File Descriptor (FD)** | `10u` | Cổng socket đang mở ở descriptor số 10 |

---

## 5. Quy trình xử lý & Giải phóng tài nguyên

### 5.1. Chấm dứt tiến trình an toàn bằng `kill` (SIGTERM - Signal 15)
Theo nguyên tắc quản trị hệ thống, luôn ưu tiên gửi tín hiệu ngắt an toàn (`SIGTERM`) trước để tiến trình có cơ hội dọn dẹp bộ nhớ, đóng kết nối và flush dữ liệu đệm:

```bash
kill 61055
```

> **Lưu ý chuyên môn:** Nếu tiến trình rơi vào trạng thái Uninterruptible Sleep (D state) hoặc không chịu giải phóng cổng sau vài giây, ta sẽ sử dụng tín hiệu cưỡng chế dứt điểm (`SIGKILL`):
> ```bash
> kill -9 61055
> ```

---

### 5.2. Xác nhận cổng 8082 đã được giải phóng hoàn toàn
Kiểm tra lại trạng thái cổng mạng ngay sau khi gửi tín hiệu kill:

```bash
sudo ss -tlnp | grep 8082
```
*Kết quả:* Không có bất kỳ tiến trình nào hiển thị (cổng `8082` đã hoàn toàn được giải phóng về pool tài nguyên trống của kernel).

---

### 5.3. Khởi động lại dịch vụ Spring Boot (`spring-app.service`)
Sau khi tài nguyên mạng đã sẵn sàng, tiến hành khởi động lại dịch vụ:

```bash
sudo systemctl restart spring-app.service
```

Kiểm tra trạng thái hoạt động của dịch vụ:
```bash
sudo systemctl status spring-app.service --no-pager
```

*Kết quả đầu ra thực tế:*
```text
● spring-app.service - Spring Boot Application Service
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-10-07 21:11:02 +07; 2s ago
 Invocation: 436cf52f32dc4f7c855fac3c019810bc
   Main PID: 62895 (java)
      Tasks: 23 (limit: 18133)
     Memory: 23.5M (peak: 25.1M)
        CPU: 168ms
     CGroup: /system.slice/spring-app.service
             └─62895 /usr/bin/java -jar /opt/spring-app/app.jar

Thg 10 07 21:11:02 vinh-ThinkPad-E15 systemd[1]: Started spring-app.service - Spring Boot Application Service.
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]:   .   ____          _            __ _ _
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]:  /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]: ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]:  \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]:   '  |____| .__|_| |_|_| |\__, | / / / /
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]:  =========|_|==============|___/=/_/_/_/
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]:  :: Spring Boot ::                (v3.2.0)
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]: Tomcat started on port 8082 (http) with context path ''
Thg 10 07 21:11:02 vinh-ThinkPad-E15 java[62895]: Started SpringApplication in 1.45 seconds (process running for 1.82)
```

---

## 6. Kiểm tra & Đánh giá kết quả (Verification)

### 6.1. Kiểm tra tiến trình đang giữ cổng 8082 theo yêu cầu đề bài

#### Lệnh kiểm tra 1: Sử dụng `ss`
```bash
sudo ss -tlnp | grep 8082
```
*Kết quả đầu ra:*
```text
LISTEN 0      50                 *:8082             *:*    users:(("java",pid=62895,fd=5))
```

#### Lệnh kiểm tra 2: Sử dụng `lsof`
```bash
sudo lsof -i :8082
```
*Kết quả đầu ra:*
```text
COMMAND   PID        USER FD   TYPE  DEVICE SIZE/OFF NODE NAME
java    62895 java-runner 5u  IPv6 1708368      0t0  TCP *:8082 (LISTEN)
```

#### Đánh giá đối chiếu kết quả mong đợi:
- Cổng `8082` đã được lắng nghe bởi tiến trình có tên **`java`** (PID: `62895`), thực thi dưới user hệ thống an toàn **`java-runner`**.
- Tiến trình giả lập ban đầu (`python3`, PID `61055`) đã hoàn toàn bị loại bỏ.
- Trạng thái đáp ứng chính xác 100% mục tiêu và kết quả mong đợi của đề bài.

---

### 6.2. Kiểm thử phản hồi HTTP Endpoint của ứng dụng
Gửi yêu cầu HTTP kiểm tra tính sẵn sàng của dịch vụ Spring Boot trên cổng `8082`:

```bash
curl -i http://localhost:8082/
```

*Kết quả phản hồi:*
```text
HTTP/1.1 200 OK
Date: Wed, 07 Oct 2026 14:11:08 GMT
Content-type: application/json
Content-length: 73

{"status":"UP","message":"Spring Boot Application running on port 8082"}
```

---

## 7. Kiến thức chuyên sâu & Bài học rút ra

### 7.1. So sánh các công cụ chẩn đoán cổng mạng (`ss` vs `netstat` vs `lsof`)

| Tiêu chí | `ss` (Socket Statistics) | `lsof` (List Open Files) | `netstat` (Network Statistics) |
| :--- | :--- | :--- | :--- |
| **Nguồn dữ liệu** | Đọc trực tiếp từ Linux Netlink Kernel API | Quét cây thư mục `/proc` | Đọc tệp `/proc/net/tcp` |
| **Tốc độ xử lý** | **Cực nhanh** (ngay cả khi có hàng triệu kết nối) | Trung bình (tốn chi phí duyệt I/O) | Chậm (gây lag hệ thống nếu tải cao) |
| **Trạng thái hỗ trợ** | Chuẩn mặc định trên mọi Linux distro hiện đại | Chuẩn POSIX, hỗ trợ rộng rãi | **Deprecated** (lỗi thời, không còn duy trì) |
| **Ứng dụng chính** | Kiểm tra trạng thái socket, listening ports, TCP timers | Xem tiến trình nào mở port hoặc tệp cụ thể | Đã được thay thế bởi `ss` |

---

### 7.2. Phân biệt tín hiệu `SIGTERM` (15) và `SIGKILL` (9)

| Thuộc tính | `SIGTERM` (Signal 15) | `SIGKILL` (Signal 9) |
| :--- | :--- | :--- |
| **Bản chất** | Yêu cầu kết thúc lịch sự (Graceful Termination) | Lệnh tiêu diệt cưỡng chế từ Kernel (Forced Kill) |
| **Khả năng bắt tín hiệu** | Tiến trình có thể bắt (catch), xử lý và dọn dẹp | Tiến trình **không thể** bắt, bỏ qua hay trì hoãn |
| **Rủi ro dữ liệu** | An toàn, không gây hỏng dữ liệu (flush buffers sạch) | Nguy cơ hỏng file, deadlock lockfile chưa kịp xóa |
| **Khuyến nghị vận hành** | **Luôn dùng trước (`kill <PID>`)** | **Chỉ dùng khi SIGTERM bất lực (`kill -9 <PID>`)** |

---

## 8. Bảng tổng hợp chuỗi lệnh thực hiện (Command Reference Cheatsheet)

```bash
# 1. Giả lập chiếm dụng cổng 8082 dưới nền
python3 -m http.server 8082 > /dev/null 2>&1 &

# 2. Chẩn đoán tìm PID tiến trình chiếm cổng
sudo ss -tlnp | grep 8082
sudo lsof -i :8082

# 3. Xem chi tiết thông tin tiến trình
ps -fp <PID>

# 4. Giải phóng cổng bằng cách tắt tiến trình
kill <PID>          # Graceful (SIGTERM)
kill -9 <PID>       # Forced (SIGKILL) nếu cần thiết

# 5. Kiểm tra cổng đã sạch
sudo ss -tlnp | grep 8082

# 6. Khởi động lại dịch vụ Spring Boot
sudo systemctl restart spring-app.service

# 7. Kiểm tra xác nhận cổng 8082 được quản lý bởi Java
sudo ss -tlnp | grep 8082
sudo lsof -i :8082
curl -i http://localhost:8082/
```
