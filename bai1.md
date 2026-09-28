# BÁO CÁO THỰC HÀNH: TỔNG HỢP CẤU HÌNH HẠ TẦNG WEB MULTI-SITE

Báo cáo chi tiết quá trình khởi tạo môi trường Linux, cài đặt Docker Compose, thiết lập 5 dịch vụ lõi và cấu hình Reverse Proxy Nginx chạy 2 website độc lập qua Cloudflare Tunnel.

---

## 1. PHÂN TÍCH & LỰA CHỌN MÔI TRƯỜNG GIẢ LẬP LINUX OS

So sánh 4 giải pháp giả lập Linux phổ biến:

* **WSL 2 (Windows Subsystem for Linux):** Khởi động tức thì, tiêu tốn ít RAM/CPU, tích hợp sâu vào hệ thống tệp Windows. **(Lựa chọn tối ưu nhất cho bài tập)**.
* **VirtualBox:** Miễn phí, hỗ trợ giao diện GUI đầy đủ nhưng tốn nhiều tài nguyên.
* **VMware Workstation:** Hiệu năng ảo hóa cao, quản lý card mạng tốt nhưng đòi hỏi bản quyền.
* **Hyper-V:** Tích hợp sẵn trên Windows Pro, hiệu năng cao nhưng dễ xung đột với các trình giả lập khác.

### Câu lệnh khởi tạo WSL 2 (PowerShell Admin):
```powershell
wsl --install -d Ubuntu
```

---

## 2. DỊCH VỤ VÀ CÂY THƯ MỤC DỰ ÁN

### 2.1 Tác dụng của các dịch vụ (Services)

| Tên Dịch Vụ | Hình ảnh Container | Vai Trò & Tác Dụng |
| :--- | :--- | :--- |
| **Nginx** | `nginx:alpine` | **Reverse Proxy & Web Server:** Tiếp nhận traffic HTTP (Cổng 80), định tuyến yêu cầu dựa theo tên miền (`server_name`) về thư mục web tương ứng hoặc chuyển tiếp API sang Node-RED. |
| **Node-RED** | `nodered/node-red` | **IoT Workflow Platform:** Chạy ứng dụng lập trình dòng chảy (Flow-based) xử lý logic và dữ liệu ở cổng 1880. |
| **MariaDB** | `mariadb:11` | **Database Server:** Quản trị cơ sở dữ liệu quan hệ SQL phục vụ các ứng dụng web. |
| **phpMyAdmin**| `phpmyadmin` | **Database Management GUI:** Giao diện đồ họa web giúp thao tác, quản lý MariaDB trực quan ở cổng 8080. |
| **Cloudflared**| `cloudflare/cloudflared` | **Secure HTTPS Tunnel:** Kết nối an toàn máy local ra Internet qua Cloudflare Zero Trust mà không cần Mở Port (Port Forwarding) hay có IP Tĩnh. |

---

### 2.2 Cấu trúc cây thư mục (Directory Structure)

```text
~/webkmt/docker/
├── docker-compose.yml       # File định nghĩa 5 dịch vụ Docker
├── nginx/
│   └── conf.d/
│       ├── web1.conf        # Cấu hình Virtual Host cho Website 1 & Proxy Node-RED
│       └── web2.conf        # Cấu hình Virtual Host cho Website 2
├── web/
│   ├── site1/
│   │   └── index.html       # Mã nguồn HTML trang Web 1
│   └── site2/
│       └── index.html       # Mã nguồn HTML trang Web 2
├── nodered_data/            # Dữ liệu lưu trữ bền vững của Node-RED
└── mariadb_data/            # Dữ liệu lưu trữ bền vững của MariaDB
```

---

## 3. TỔNG HỢP CÁC CÂU LỆNH ĐÃ THỰC THI

### Bước 3.1: Cài đặt Docker & Docker Compose trên Ubuntu
```bash
# 1. Cập nhật hệ thống
sudo apt update && sudo apt upgrade -y

# 2. Cài đặt Docker tự động từ script trang chủ
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 3. Phân quyền chạy Docker cho User hiện tại & cài gói hỗ trợ nhóm
sudo apt install util-linux-extra psmisc -y
sudo usermod -aG docker $USER
newgrp docker

# 4. Kiểm tra phiên bản Docker
docker --version && docker compose version
```

### Bước 3.2: Khởi tạo thư mục & Phân quyền
```bash
# Tạo cấu trúc thư mục
mkdir -p ~/webkmt/docker/web/site1
mkdir -p ~/webkmt/docker/web/site2
mkdir -p ~/webkmt/docker/nginx/conf.d
mkdir -p ~/webkmt/docker/nodered_data
mkdir -p ~/webkmt/docker/mariadb_data

# Sửa quyền thư mục dữ liệu cho UID 1000 (Node-RED)
sudo chown -R 1000:1000 ~/webkmt/docker/nodered_data
```

### Bước 3.3: Tạo các file mã nguồn và cấu hình Nginx

* **Tạo file HTML mẫu:**
```bash
# Web 1
cat << 'EOF' > ~/webkmt/docker/web/site1/index.html
<!DOCTYPE html>
<html><head><meta charset="UTF-8"><title>Web 1</title></head>
<body><h1>Chào mừng tới Website 1 (web1.phuongkmt.id.vn)</h1></body></html>
EOF

# Web 2
cat << 'EOF' > ~/webkmt/docker/web/site2/index.html
<!DOCTYPE html>
<html><head><meta charset="UTF-8"><title>Web 2</title></head>
<body><h1>Chào mừng tới Website 2 (web2.phuongkmt.id.vn)</h1></body></html>
EOF
```

* **Tạo file cấu hình Virtual Hosts Nginx:**
```bash
# Virtual Host 1 (web1.conf)
cat << 'EOF' > ~/webkmt/docker/nginx/conf.d/web1.conf
server {
    listen 80;
    server_name web1.phuongkmt.id.vn;

    location / {
        root /usr/share/nginx/html/site1;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://nodered:1880/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
EOF

# Virtual Host 2 (web2.conf)
cat << 'EOF' > ~/webkmt/docker/nginx/conf.d/web2.conf
server {
    listen 80;
    server_name web2.phuongkmt.id.vn;

    location / {
        root /usr/share/nginx/html/site2;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }
}
EOF
```

### Bước 3.4: Tạo file `docker-compose.yml`
```bash
cat << 'EOF' > ~/webkmt/docker/docker-compose.yml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./web:/usr/share/nginx/html:ro
    depends_on:
      - nodered
    restart: unless-stopped

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    ports:
      - "1880:1880"
    volumes:
      - ./nodered_data:/data
    restart: unless-stopped

  mariadb:
    image: mariadb:11
    container_name: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: root123456
      MYSQL_DATABASE: webmt
      MYSQL_USER: webuser
      MYSQL_PASSWORD: web123456
    volumes:
      - ./mariadb_data:/var/lib/mysql
    restart: unless-stopped

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    ports:
      - "8080:80"
    environment:
      PMA_HOST: mariadb
      PMA_PORT: 3306
    depends_on:
      - mariadb
    restart: unless-stopped

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token eyJhSW...
EOF
```

### Bước 3.5: Các lệnh giải phóng cổng & Quản lý Container
```bash
# Cưỡng chế dừng & giải phóng các cổng kẹt (80, 1880, 8080)
sudo fuser -k 8080/tcp 80/tcp 1880/tcp

# Xóa container cũ
docker rm -f $(docker ps -aq)

# Khởi chạy toàn bộ hệ thống ở chế độ ngầm
cd ~/webkmt/docker
docker compose up -d

# Kiểm tra trạng thái các service
docker compose ps

# Kiểm tra cú pháp file cấu hình Nginx
docker compose exec nginx nginx -t

# Kiểm tra điều hướng Nginx nội bộ
curl -i -H "Host: web1.phuongkmt.id.vn" http://localhost
curl -i -H "Host: web2.phuongkmt.id.vn" http://localhost
```

---

## 4. CẤU HÌNH CLOUDFLARE ZERO TRUST & KẾT QUẢ

### Cấu hình Public Hostname trong Cloudflare Tunnel:
1. `web1.phuongkmt.id.vn` $\rightarrow$ Service: `HTTP` | URL: `nginx:80`
2. `web2.phuongkmt.id.vn` $\rightarrow$ Service: `HTTP` | URL: `nginx:80`

### Kết quả truy cập thực tế qua Internet:
* **Website 1:** `https://web1.phuongkmt.id.vn` (Hiện đúng nội dung `site1`)
* **Website 2:** `https://web2.phuongkmt.id.vn` (Hiện đúng nội dung `site2`)







routes
<img width="960" height="1029" alt="image" src="https://github.com/user-attachments/assets/30cbb5ca-44c3-4446-af3c-948a3303d7e9" />
