---
description: dùng để note những kiến thức về linux rút ra trong quá trình học và practice
---

# Linux Notes — Bảng Quản Lý Tập Trung

| STT | Chủ đề | Mục đích | Lệnh / Cấu hình chính | Cơ chế cốt lõi | Lưu ý quan trọng (Gotchas) |
| :---: | :--- | :--- | :--- | :--- | :--- |
| **1** | **`getent`** | Tra cứu danh bạ (user, group, host) thống nhất từ cả nguồn local lẫn máy chủ mạng (LDAP, AD). | `getent passwd [user]`<br>`getent group [group]`<br>`getent hosts [domain]` | Truy vấn qua điều hướng `/etc/nsswitch.conf`, gom kết quả từ file nội bộ (`/etc/*`) và máy chủ mạng thay vì chỉ đọc file tĩnh. | `cat /etc/passwd` chỉ thấy tài khoản local, bị mù tịt về tài khoản mạng enterprise. |
| **2** | **Thứ tự kiểm tra quyền**<br>(Owner vs Group) | Xác định quyền truy cập thực tế của người dùng khi thao tác trên file/thư mục. | `ls -l`<br>`chmod <mode> <file>` | Kiểm tra tuần tự độc lập, **không cộng dồn**:<br>1. Trùng UID → dùng bộ **Owner**<br>2. Trùng GID → dùng bộ **Group**<br>3. Không khớp → dùng bộ **Other** | Nếu user là Owner thì chỉ áp dụng quyền Owner, kể cả khi group của user đó có quyền lớn hơn. |
| **3** | **`setgid` thư mục**<br>(`chmod 2775`) | Thiết lập thư mục chia sẻ nhóm (collaborative shared folder) cho nhiều người cùng làm việc. | `chgrp <group> <dir>`<br>`chmod 2775 <dir>` | Bit `2` (setgid) ép mọi file/thư mục con mới tạo **tự động thừa kế group của thư mục cha** (thay vì lấy group chính của người tạo). | Group phải có quyền ghi (`w`) (dùng `2775`, không dùng `2755`) thì thành viên trong group mới tạo được file bên trong. |
| **4** | **SSH Chroot SFTP Jail** | Cách ly tuyệt đối tài khoản: chỉ cho truyền tệp SFTP trong thư mục chỉ định, cấm mở terminal và cấm forward mạng. | Cấu hình `/etc/ssh/sshd_config`:<br>`Match User <user>`<br>&nbsp;&nbsp;`ChrootDirectory <dir>`<br>&nbsp;&nbsp;`ForceCommand internal-sftp`<br>&nbsp;&nbsp;`AllowTcpForwarding no`<br>&nbsp;&nbsp;`X11Forwarding no`<br><br>Kiểm tra & reload:<br>`sudo sshd -t`<br>`sudo systemctl restart sshd` | Biến thư mục chỉ định thành gốc ảo (`/`) đối với user; ép phiên làm việc qua internal-sftp và chặn các kênh chuyển tiếp dữ liệu. | Thư mục `ChrootDirectory` **bắt buộc** do `root:root` sở hữu và có quyền `755`. Nếu để user sở hữu, `sshd` sẽ từ chối kết nối ngay lập tức. |
| **5** | **`docker run` cơ bản** | Tạo, cấu hình và khởi chạy ứng dụng trong container biệt lập ở chế độ ngầm và mở port ra ngoài. | `docker run -d --name <tên> -p <host>:<container> <image>`<br><br>*Ví dụ:*<br>`docker run -d --name my-nginx -p 80:80 nginx` | Khởi tạo container từ image:<br>• `-d` (`--detach`): Chạy ngầm, nhả terminal<br>• `--name`: Đặt tên định danh container<br>• `-p`: Ánh xạ cổng host vào container | Cú pháp cổng là `<Host_Port>:<Container_Port>`. Tự động pull image từ Docker Hub về nếu ở local chưa có sẵn. |

  • docker volume create: Lệnh tạo một vùng lưu trữ dữ liệu (Volume) độc lập trên máy chủ do Docker quản lý.
  • n8n_data: Tên của volume. Việc tạo volume giúp lưu trữ toàn bộ workflows, credentials và thông tin cấu hình
  của n8n. Khi container bị dừng, xóa hoặc nâng cấp lên phiên bản mới, dữ liệu của bạn sẽ không bị mất.


Tôi đã hoàn thành việc triển khai kiến trúc tách rời từng file compose theo đúng lựa chọn của bạn tại thư mục
  /users.
  Dưới đây là cấu trúc thư mục, chi tiết các câu lệnh đã chạy và giải thích từng bước:
  ──────
  ### 1. Cấu trúc thư mục tại /users
    /users/
    ├── n8n/
    │   └── docker-compose.yml        # Quản lý riêng dịch vụ n8n
    └── nginx-proxy/
        ├── docker-compose.yml        # Quản lý riêng Nginx Gateway
        └── conf.d/
            └── n8n.conf              # Cấu hình Reverse Proxy trỏ đến container n8n
  ──────
  ### 2. Các câu lệnh đã thực hiện và giải thích chi tiết
  #### Bước 1: Khởi tạo Docker Network dùng chung và dọn dẹp container cũ

    docker network create proxy-network && docker stop my-nginx n8n && docker rm my-nginx n8n

  • Giải thích:
      • docker network create proxy-network: Tạo mạng bridge có tên proxy-network. Mạng này đóng vai trò làm cầu
      nối để các container ở các file compose khác nhau (Nginx và n8n) có thể tìm thấy và giao tiếp với nhau bằng
      tên container.
      • docker stop my-nginx n8n: Dừng 2 container chạy đơn lẻ cũ.
      • docker rm my-nginx n8n: Xóa 2 container cũ để giải phóng tài nguyên. (Volume n8n_data được giữ nguyên vẹn
      100%, không bị ảnh hưởng).

  ──────
  #### Bước 2: Tạo cấu hình cho n8n (/users/n8n/docker-compose.yml)

  File này khai báo container n8n gia nhập vào mạng proxy-network và gắn volume dữ liệu n8n_data:
    services:
      n8n:
        image: n8nio/n8n:latest
        container_name: n8n
        restart: unless-stopped
        networks:
          - proxy-network
        environment:
          - N8N_PORT=5678
          - N8N_PROTOCOL=http
          - NODE_ENV=production
          - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
        volumes:
          - n8n_data:/home/node/.n8n

    networks:
      proxy-network:
        external: true

    volumes:
      n8n_data:
        external: true
  • Điểm nổi bật: Không cần map port 5678:5678 ra bên ngoài internet nữa, giúp bảo vệ an toàn cho n8n; chỉ có
  Nginx trong mạng nội bộ mới gọi được vào n8n.
  ──────
  #### Bước 3: Tạo cấu hình cho Nginx Proxy (/users/nginx-proxy/)
  File cấu hình chuyển tiếp /users/nginx-proxy/conf.d/n8n.conf:
    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://n8n:5678;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
  • Giải thích: proxy_pass http://n8n:5678 sử dụng trực tiếp tên container n8n làm hostname. Các dòng Upgrade và
  Connection 'upgrade' giúp n8n hoạt động tốt với WebSocket (giao tiếp realtime trên giao diện).

  File /users/nginx-proxy/docker-compose.yml:
    services:
      nginx:
        image: nginx:latest
        container_name: nginx-proxy
        restart: unless-stopped
        networks:
          - proxy-network
        ports:
          - "80:80"
        volumes:
          - ./conf.d:/etc/nginx/conf.d:ro

    networks:
      proxy-network:
        external: true
  • Giải thích: Nginx mở cổng 80:80 ra ngoài máy chủ để đón nhận mọi truy cập web và mount thư mục conf.d vào
  container.
  ──────
  #### Bước 4: Khởi chạy từng dịch vụ bằng Docker Compose
    cd /users/n8n && docker compose up -d
    cd /users/nginx-proxy && docker compose up -d

  • Giải thích:
      • docker compose up -d: Đọc file docker-compose.yml tại thư mục hiện tại, tự động tạo container và khởi chạy
      ngầm trong nền (-d).

  ──────
  #### Bước 5: Kiểm tra kết quả

    docker ps
    curl -s -o /dev/null -w "%{http_code}\n" http://localhost:80

  • Giải thích:
      • docker ps: Xác nhận cả 2 container nginx-proxy (port 80) và n8n (nội bộ) đều đang chạy Up.
      • curl: Kiểm tra truy vấn HTTP vào port 80 qua Nginx, trả về mã 200 và nội dung n8n Editor UI đầy đủ.

  ──────
  ### 3. Cách truy cập và quản lý

  • Truy cập giao diện n8n: Bây giờ bạn chỉ cần mở trình duyệt và gõ thẳng địa chỉ IP (cổng 80 mặc định, không cần
  gõ :5678 nữa):
  👉 http://103.252.137.170
  • Quản lý riêng n8n:
    cd /users/n8n
    docker compose restart    # Khởi động lại n8n
    docker compose logs -f    # Xem log n8n

  • Quản lý riêng Nginx (khi sau này bạn muốn thêm website mới):
    cd /users/nginx-proxy
    # Thêm file config mới vào thư mục conf.d/ rồi chạy:
    docker compose exec nginx nginx -s reload  # Tải lại cấu hình không cần downtime

Lỗi này xuất hiện do n8n phiên bản mới mặc định bật tính năng Secure Cookie (chỉ cho phép lưu cookie đăng nhập
  qua kết nối bảo mật HTTPS). Vì hiện tại bạn đang truy cập qua HTTP thường bằng địa chỉ IP
  (http://103.252.137.170) nên n8n chặn không cho đăng nhập.

  Tôi đã khắc phục vấn đề này. Dưới đây là chi tiết thay đổi và câu lệnh đã thực hiện:
  ──────
  ### 1. Thay đổi cấu hình tại /users/n8n/docker-compose.yml

  Đã thêm biến môi trường N8N_SECURE_COOKIE=false vào phần environment:

        environment:
          - N8N_SECURE_COOKIE=false      # Cho phép hoạt động qua HTTP thường (IP)
          - N8N_PORT=5678
          - N8N_PROTOCOL=http
          - NODE_ENV=production
          - GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
  ──────
  ### 2. Câu lệnh đã chạy và giải thích

    cd /users/n8n && docker compose up -d

  Giải thích:

  • cd /users/n8n: Di chuyển vào thư mục quản lý của n8n.
  • docker compose up -d: Docker Compose tự động phát hiện file cấu hình vừa được cập nhật thêm biến môi trường,
  nó sẽ chỉ tái tạo lại container n8n (Container n8n Recreate) để áp dụng ngay cấu hình mới mà không làm mất bất
  kỳ dữ liệu nào trong volume.
  ──────
  ### 3. Kiểm tra lại

  Bây giờ bạn hãy làm mới trang web trên trình duyệt (nhấn Ctrl + F5 trên Windows hoặc Cmd + Shift + R trên Mac):

  👉 http://103.252.137.170

  Giao diện khởi tạo tài khoản ban đầu của n8n sẽ hiển thị bình thường mà không còn bị cảnh báo Secure Cookie nữa!