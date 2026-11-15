# Hướng Dẫn Chi Tiết: Quá Trình Thiết Lập SSL / HTTPS Cho Domain `taphoacongnghe.duckdns.org`

Tài liệu này ghi lại toàn bộ nguyên nhân, phân tích kiến trúc hệ thống, quy trình xử lý và danh sách đầy đủ các câu lệnh Linux đã thực thi để giải quyết lỗi **"Not Secure"** trên trang web của bạn.

---

## 1. Bản Chất Vấn Đề (Nguyên Nhân "Not Secure")

* **Giao thức HTTP (Cổng 80)**: Trước đó, trang web chỉ chạy trên cổng 80 không mã hóa. Dữ liệu gửi/nhận giữa người dùng và máy chủ là dạng văn bản thô. Trình duyệt (Chrome, Safari, Firefox,...) sẽ gắn nhãn cảnh báo **"Not Secure" (Không bảo mật)** để cảnh báo người dùng.
* **Giao thức HTTPS (Cổng 443)**: Để trình duyệt hiển thị biểu tượng **ổ khóa bảo mật**, toàn bộ luồng truyền tải cần được mã hóa qua SSL/TLS bằng một chứng chỉ số hợp lệ được ký bởi tổ chức CA uy tín (ở đây sử dụng **Let's Encrypt**).

---

## 2. Khảo Sát Hệ Thống Trước Khi Xử Lý

Tôi đã kiểm tra máy chủ và phát hiện các thành phần:
* **Tên miền**: `taphoacongnghe.duckdns.org` đã trỏ đúng về địa chỉ IP của VPS (`103.252.137.170`).
* **Hạ tầng dịch vụ**: Hệ thống đang chạy bằng **Docker Compose** tại thư mục `/users/`:
  - `nginx-proxy`: Đóng vai trò Reverse Proxy (nhưng trước đó chỉ mở cổng 80).
  - `n8n`: Ứng dụng n8n chạy ở cổng nội bộ 5678.
  - `postgres`: Cơ sở dữ liệu cho n8n.

---

## 3. Toàn Bộ Lệnh Đã Chạy & Các Bước Thực Hiện

### Bước 1: Khảo sát địa chỉ IP, trạng thái cổng mạng và DNS
Kiểm tra xem tên miền đã trỏ đúng về VPS chưa và cổng nào đang hoạt động:
```bash
# Kiểm tra IP công khai và kiểm tra bản ghi DNS của DuckDNS
curl -s ifconfig.me
dig +short taphoacongnghe.duckdns.org

# Kiểm tra các cổng đang lắng nghe (listening)
ss -tulpn

# Kiểm tra các container Docker đang hoạt động
docker ps -a
```

---

### Bước 2: Cài đặt công cụ cấp chứng chỉ SSL (Certbot)
Cài đặt `certbot` trực tiếp trên hệ điều hành Ubuntu:
```bash
apt-get update -qq && apt-get install -y -qq certbot
```

---

### Bước 3: Xin cấp chứng chỉ SSL miễn phí từ Let's Encrypt
Để Let's Encrypt cấp chứng chỉ qua cơ chế HTTP-01 challenge độc lập, tạm dừng Nginx vài giây để Certbot dùng cổng 80, sau đó khởi động lại:
```bash
# Tạm dừng Nginx để giải phóng cổng 80
docker stop nginx-proxy

# Xin cấp chứng chỉ SSL cho tên miền taphoacongnghe.duckdns.org
certbot certonly --standalone \
  -d taphoacongnghe.duckdns.org \
  --non-interactive \
  --agree-tos \
  --register-unsafely-without-email

# Bật lại Nginx
docker start nginx-proxy
```

> [!NOTE]
> Chứng chỉ sau khi cấp thành công được lưu tại:
> - **File chứng chỉ (Certificate)**: `/etc/letsencrypt/live/taphoacongnghe.duckdns.org/fullchain.pem`
> - **Khóa bí mật (Private Key)**: `/etc/letsencrypt/live/taphoacongnghe.duckdns.org/privkey.pem`

---

### Bước 4: Thiết lập tự động gia hạn chứng chỉ (Auto-Renewal)
Chứng chỉ Let's Encrypt có thời hạn 90 ngày. Để máy chủ tự động gia hạn mà không bao giờ bị gián đoạn hay phải thao tác bằng tay:
```bash
# Tạo thư mục hook cho Certbot
mkdir -p /etc/letsencrypt/renewal-hooks/pre /etc/letsencrypt/renewal-hooks/post

# Tạo script dừng Nginx trước khi gia hạn
cat << 'EOF' > /etc/letsencrypt/renewal-hooks/pre/stop-nginx.sh
#!/bin/bash
docker stop nginx-proxy
EOF

# Tạo script bật lại Nginx sau khi gia hạn xong
cat << 'EOF' > /etc/letsencrypt/renewal-hooks/post/start-nginx.sh
#!/bin/bash
docker start nginx-proxy
EOF

# Cấp quyền thực thi cho 2 script
chmod +x /etc/letsencrypt/renewal-hooks/pre/stop-nginx.sh /etc/letsencrypt/renewal-hooks/post/start-nginx.sh

# Chạy thử nghiệm giả lập gia hạn tự động (dry-run)
certbot renew --dry-run
```

---

### Bước 5: Cấu hình Nginx Docker mở cổng 443 và nạp chứng chỉ SSL

Chỉnh sửa file [/users/nginx-proxy/docker-compose.yml](file:///users/nginx-proxy/docker-compose.yml):
* Mở thêm cổng `443:443`.
* Gắn volume `- /etc/letsencrypt:/etc/letsencrypt:ro` để container Nginx đọc được chứng chỉ từ máy chủ host.

Nội dung file sau khi sửa:
```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    restart: unless-stopped
    networks:
      - proxy-network
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro

networks:
  proxy-network:
    external: true
```

---

### Bước 6: Cấu hình file Nginx Server ([/users/nginx-proxy/conf.d/n8n.conf](file:///users/nginx-proxy/conf.d/n8n.conf))

Cấu hình 2 nhiệm vụ:
1. **Khối cổng 80 (HTTP)**: Tự động chuyển hướng (Redirect 301) tất cả yêu cầu sang `https://`.
2. **Khối cổng 443 (HTTPS)**: Nhận chứng chỉ SSL, kích hoạt HTTP/2 và chuyển tiếp dữ liệu đến n8n qua WebSocket/SSE.

Nội dung file cấu hình:
```nginx
server {
    listen 80;
    server_name taphoacongnghe.duckdns.org;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name taphoacongnghe.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/taphoacongnghe.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/taphoacongnghe.duckdns.org/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    client_max_body_size 50M;

    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

### Bước 7: Cập nhật biến môi trường n8n

Chỉnh sửa file [/users/n8n/docker-compose.yml](file:///users/n8n/docker-compose.yml) để n8n hoạt động đồng bộ với HTTPS và tạo link Webhook chuẩn:
* Thêm `N8N_HOST=taphoacongnghe.duckdns.org`
* Chuyển `N8N_PROTOCOL=https`
* Thêm `WEBHOOK_URL=https://taphoacongnghe.duckdns.org/`

---

### Bước 8: Áp dụng thay đổi cho các Docker Container

Chạy lệnh để Docker tạo lại container với các cổng và cấu hình mới:
```bash
# Tái tạo Nginx Proxy với cổng 443 mới
cd /users/nginx-proxy && docker compose up -d --force-recreate

# Tái tạo n8n với biến môi trường HTTPS mới
cd /users/n8n && docker compose up -d
```

---

### Bước 9: Kiểm tra và xác thực kết quả

Chạy lệnh kiểm tra phản hồi từ máy chủ:
```bash
# Kiểm tra chuyển hướng HTTP -> HTTPS (kết quả trả về mã 301)
curl -I http://taphoacongnghe.duckdns.org

# Kiểm tra kết nối bảo mật HTTPS (kết quả trả về HTTP/2 200 OK)
curl -sSL -I https://taphoacongnghe.duckdns.org
```

---

## 4. Các Lệnh Hữu Ích Dành Cho Bạn Trong Tương Lai

* **Kiểm tra thời hạn chứng chỉ SSL**:
  ```bash
  certbot certificates
  ```
* **Kiểm tra log của Nginx Proxy**:
  ```bash
  docker logs -f nginx-proxy
  ```
* **Kiểm tra log của n8n**:
  ```bash
  docker logs -f n8n
  ```
* **Khởi động lại toàn bộ dịch vụ (nếu cần)**:
  ```bash
  cd /users/nginx-proxy && docker compose restart
  cd /users/n8n && docker compose restart
  ```
