# Bài 4: Cấu hình Reverse Proxy Nginx cho Ứng dụng Spring Boot

## Mục tiêu
Cấu hình Nginx Server Block làm Reverse Proxy, phân chia lưu lượng:
- **`/`** → Phục vụ trang tĩnh từ `/var/www/html/`
- **`/api/`** → Chuyển tiếp đến Spring Boot backend tại cổng **8082**

---

## Cấu trúc thư mục nộp bài

```
ex4/
├── spring-proxy.conf   # Cấu hình Nginx virtual host
├── index.html          # Trang tĩnh đặt tại /var/www/html/
└── README.md           # Báo cáo kết quả
```

---

## Hướng dẫn triển khai

### Bước 1: Tạo thư mục và trang tĩnh

```bash
# Tạo thư mục web root (thường đã tồn tại)
sudo mkdir -p /var/www/html

# Copy file index.html vào web root
sudo cp index.html /var/www/html/index.html

# Đặt quyền sở hữu cho Nginx
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

### Bước 2: Cài đặt file cấu hình Nginx

```bash
# Copy file cấu hình vào sites-available
sudo cp spring-proxy.conf /etc/nginx/sites-available/spring-proxy.conf
```

### Bước 3: Kích hoạt cấu hình (tạo symlink)

```bash
# Xóa cấu hình mặc định nếu cần (tùy chọn)
# sudo rm /etc/nginx/sites-enabled/default

# Tạo liên kết mềm sang sites-enabled
sudo ln -s /etc/nginx/sites-available/spring-proxy.conf \
           /etc/nginx/sites-enabled/spring-proxy.conf
```

### Bước 4: Kiểm tra cú pháp và reload

```bash
# Kiểm tra cú pháp cấu hình
sudo nginx -t

# Reload Nginx để áp dụng cấu hình mới
sudo systemctl reload nginx
```

---

## Kết quả kiểm tra

### 1. Kiểm tra cú pháp Nginx

```
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

✅ **PASS** — Cú pháp hợp lệ, cấu hình không có lỗi.

<!-- 📸 DÁN ẢNH 1: Chụp màn hình terminal kết quả lệnh `sudo nginx -t` vào đây -->
![Kết quả nginx -t](./screenshots/nginx-t.png)

### 2. Kiểm tra trang tĩnh (cổng 80)

```
$ curl -I http://localhost/
HTTP/1.1 200 OK
Server: nginx/1.28.3 (Ubuntu)
Date: Thu, 08 Oct 2026 17:25:10 GMT
Content-Type: text/html
Content-Length: 6619
Last-Modified: Thu, 08 Oct 2026 17:23:47 GMT
Connection: keep-alive
ETag: "6ac7d1a3-19db"
Accept-Ranges: bytes
```

✅ **PASS** — HTTP 200 OK, Nginx phục vụ trang tĩnh `index.html` thành công.

<!-- 📸 DÁN ẢNH 2: Chụp màn hình terminal kết quả lệnh `curl -I http://localhost/` vào đây -->
![Kết quả curl -I localhost](./screenshots/curl-header.png)

<!-- 📸 DÁN ẢNH 3: Chụp màn hình trình duyệt mở http://localhost/ vào đây -->
![Trang web trên trình duyệt](./screenshots/browser-localhost.png)

### 3. Kiểm tra API Reverse Proxy

```bash
# Kiểm tra endpoint health của Spring Boot qua Nginx
curl -I http://localhost/api/health
```

> **Lưu ý:** Nếu Spring Boot chưa chạy, lệnh trên sẽ trả về `502 Bad Gateway`.  
> Khi Spring Boot đang chạy ở cổng 8082, kết quả sẽ là `200 OK` hoặc mã tương ứng.

---

## Giải thích cấu hình

### Sơ đồ luồng request

```
Client (Browser/curl)
        │
        ▼
  Nginx (Port 80)
        │
        ├─── location /      ──────────────►  /var/www/html/index.html
        │    (Static files)
        │
        └─── location /api/  ──────────────►  Spring Boot (127.0.0.1:8082)
             (Reverse Proxy)                  app chạy tại cổng 8082
```

### Các Proxy Header quan trọng

| Header | Giá trị | Mục đích |
|--------|---------|---------|
| `Host` | `$host` | Chuyển tiếp hostname gốc của request |
| `X-Real-IP` | `$remote_addr` | IP thực của client |
| `X-Forwarded-For` | `$proxy_add_x_forwarded_for` | Chuỗi IP qua các proxy |
| `X-Forwarded-Proto` | `$scheme` | Giao thức gốc (http/https) |

### Lý do dùng `proxy_pass http://127.0.0.1:8082/`

- Dùng **IP loopback** (`127.0.0.1`) thay vì `localhost` để tránh phân giải DNS không cần thiết và giảm độ trễ.
- Dấu `/` cuối trong `proxy_pass` giúp Nginx **loại bỏ tiền tố `/api/`** trước khi chuyển tiếp — tức là `/api/health` → `/health` ở backend.

---

## Lệnh kiểm tra nhanh

```bash
# Xem trạng thái Nginx
sudo systemctl status nginx

# Xem log truy cập real-time
sudo tail -f /var/log/nginx/access.log

# Xem log lỗi real-time
sudo tail -f /var/log/nginx/error.log

# Liệt kê symlink trong sites-enabled
ls -la /etc/nginx/sites-enabled/
```
