# Cẩm Nang Triển Khai Ứng Dụng Lên AWS Chuẩn Production (AWS Cloud Guide)

Tài liệu này ghi lại toàn bộ quy trình thực chiến từ con số 0 đưa ứng dụng full-stack (**Vira: React + Spring Boot + MySQL + Redis**) lên nền tảng đám mây **Amazon Web Services (AWS)**, kèm theo giải thích chi tiết bản chất kỹ thuật, thuật ngữ và tư duy thiết kế hạ tầng dành cho Senior Cloud/DevOps Engineer.

---

## MỤC LỤC
1. [Từ Điển Thuật Ngữ Cốt Lõi (Core Terminology)](#1-từ-điển-thuật-ngữ-cốt-lõi)
2. [Sơ Đồ Kiến Trúc Hệ Thống (Production Architecture)](#2-sơ-đồ-kiến-trúc-hệ-thống)
3. [Quy Trình Triển Khai Từng Bước (Step-by-Step Flow)](#3-quy-trình-triển-khai-từng-bước)
   - [Giai đoạn 1: Bảo mật tài khoản & An toàn chi phí](#giai-đoạn-1-bảo-mật-tài-khoản--an-toàn-chi-phí)
   - [Giai đoạn 2: Hạ tầng tính toán EC2 & IP tĩnh Elastic IP](#giai-đoạn-2-hạ-tầng-tính-toán-ec2--ip-tĩnh-elastic-ip)
   - [Giai đoạn 3: Phân giải tên miền (DNS Resolution)](#giai-đoạn-3-phân-giải-tên-miền-dns-resolution)
   - [Giai đoạn 4: Kết nối SSH & Tối ưu Linux OS](#giai-đoạn-4-kết-nối-ssh--tối-ưu-linux-os)
   - [Giai đoạn 5: Reverse Proxy Caddy & Cấp chứng chỉ SSL HTTPS](#giai-đoạn-5-reverse-proxy-caddy--cấp-chứng-chỉ-ssl-https)
   - [Giai đoạn 6: Docker Compose Production & Ứng dụng Backend](#giai-đoạn-6-docker-compose-production--ứng-dụng-backend)
   - [Giai đoạn 7: Phân phối Frontend & Tự động hóa CI/CD](#giai-đoạn-7-phân-phối-frontend--tự-động-hóa-cicd)
4. [Các Lỗi Kinh Điển Thường Gặp & Cách Khắc Phục](#4-các-lỗi-kinh-điển-thường-gặp--cách-khắc-phục)
5. [Checklist Cho Các Dự Án Tiếp Theo](#5-checklist-cho-các-dự-án-tiếp-theo)

---

## 1. TỪ ĐIỂN THUẬT NGỮ CỐT LÕI

| Thuật ngữ | Ý nghĩa thực tế | Vai trò trong dự án |
| :--- | :--- | :--- |
| **Root User** | Tài khoản chủ cao nhất của AWS, liên kết trực tiếp với thẻ thanh toán và email đăng ký. | Dùng để thanh toán và tạo tài khoản quản trị IAM. **Không dùng để vận hành hàng ngày**. |
| **IAM (Identity and Access Management)** | Dịch vụ phân quyền người dùng và dịch vụ trên AWS. | Tạo IAM User với quyền `AdministratorAccess` giúp làm việc an toàn, tuân thủ Best Practices. |
| **AWS Region** | Vùng địa lý đặt trung tâm dữ liệu vật lý của AWS (ví dụ: `ap-southeast-1` Singapore, `us-east-1` N. Virginia). | Chọn `Singapore` để độ trễ mạng (latency) về Việt Nam thấp nhất (~30ms). |
| **EC2 (Elastic Compute Cloud)** | Máy chủ ảo (Virtual Private Server - VPS) chạy trên hạ tầng điện toán đám mây. | Nơi chạy Docker, Spring Boot API, MySQL và Redis. |
| **EBS (Elastic Block Store)** | Ổ đĩa cứng ảo (SSD) gắn vào EC2. | Chứa hệ điều hành Ubuntu và dữ liệu MySQL (ổ 20GB nằm trong hạn mức 30GB Free Tier). |
| **Elastic IP** | Địa chỉ IP Public tĩnh cố định của AWS không bị thay đổi khi restart máy chủ. | Tránh việc DNS bị mất kết nối mỗi khi khởi động lại EC2. |
| **Security Group** | Tường lửa ảo (Virtual Firewall) kiểm soát lưu lượng mạng vào/ra của EC2. | Mở port 22 (SSH), 80 (HTTP), 443 (HTTPS). Đóng port 8080, 3306, 6379 để bảo mật. |
| **DNS (Domain Name System)** | Cuốn "danh bạ điện thoại" của Internet, dịch tên miền chữ sang địa chỉ IP số. | Trỏ `vira.daoductam.dpdns.org` về IP AWS. |
| **A Record** | Bản ghi DNS ánh xạ trực tiếp một Tên miền -> Địa chỉ IPv4. | Trỏ subdomain về IP tĩnh của EC2. |
| **CNAME Record** | Bản ghi DNS bí danh, ánh xạ Tên miền -> Một Tên miền khác. | Dùng cho xác thực SSL trên ACM hoặc trỏ về AWS CloudFront CDN. |
| **Reverse Proxy** | Máy chủ đứng trước bảo vệ các dịch vụ nội bộ, tiếp nhận yêu cầu từ client và điều phối vào app. | **Caddy Server** tiếp nhận port 80/443, tự động xin SSL và forward vào Spring Boot port 8080. |
| **Let's Encrypt / ACME** | Tổ chức cấp chứng chỉ SSL miễn phí tự động qua giao thức ACME. | Caddy dùng giao thức này để lấy khóa xanh HTTPS mà không tốn chi phí. |
| **Swap Memory** | Vùng nhớ ảo trên ổ cứng SSD dùng để mở rộng khi RAM vật lý bị đầy. | Giúp máy `t3.micro` (1GB RAM) không bị crash khi chạy Java 21 và MySQL. |
| **OOM Killer** | Cơ chế của nhân Linux tự động tắt tiến trình ngốn nhiều RAM nhất khi hết bộ nhớ. | Tạo 2GB Swap triệt tiêu hoàn toàn nguy cơ OOM Killer. |
| **CI/CD Pipeline** | Tự động hóa tích hợp (CI) và triển khai (CD) mã nguồn lên môi trường máy chủ. | Dùng GitHub Actions: push code nhánh `main` là tự động build và deploy. |

---

## 2. SƠ ĐỒ KIẾN TRÚC HỆ THỐNG

```
[ NGƯỜI DÙNG TRÌNH DUYỆT (INTERNET) ]
                     │
         https://vira.daoductam.dpdns.org
         https://api-vira.daoductam.dpdns.org
                     │
                     ▼
      [ DIGITALPLAT DNS - A RECORD ]
        Trỏ về IP tĩnh: 18.139.187.188
                     │
                     ▼
  [ AWS EC2 INSTANCE (Singapore: ap-southeast-1) ]
  ┌────────────────────────────────────────────────────────┐
  │  AWS Security Group: Chỉ mở Inbound Port 22, 80, 443   │
  │  Ubuntu 24.04 LTS (1GB RAM + 2GB SSD Swap)             │
  │                                                        │
  │  [ DOCKER ENGINE ]                                     │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │  [ Caddy 2.9 (Reverse Proxy & Auto SSL) ]        │  │
  │  │   - Listen 80, 443                               │  │
  │  │   - Auto TLS (Let's Encrypt HTTPS)               │  │
  │  │   - Phục vụ Static Files /dist (Frontend)        │  │
  │  │   - Proxy /api/* sang vira-api:8080              │  │
  │  └─────────────────┬────────────────────────────────┘  │
  │                    │ Network Bridge nội bộ             │
  │       ┌────────────┴───────────┬──────────────────┐    │
  │       ▼                        ▼                  ▼    │
  │  [ vira-api ]             [ vira-mysql ]    [ vira-redis ]│
  │  Spring Boot 3.5 (Java 21) MySQL 8.4         Redis 7.4  │
  │  Port 8080 (Internal)      Port 3306(Hidden) Cache/Rate │
  │  Environment: .env         Data Volume: EBS  Limiter    │
  └────────────────────────────────────────────────────────┘
```

---

## 3. QUY TRÌNH TRIỂN KHAI TỪNG BƯỚC

### Giai đoạn 1: Bảo mật tài khoản & An toàn chi phí
1. **Đặt cảnh báo chi phí (AWS Budgets)**:
   - Truy cập **AWS Budgets** từ tài khoản Root.
   - Tạo **Zero-spend budget** hoặc hạn mức $1 - $3/tháng và nhập email cá nhân.
   - *Mục đích*: Bất kỳ phát sinh nào vượt hạn mức Free Tier đều được cảnh báo ngay.
2. **Tạo IAM User quản trị**:
   - Truy cập **IAM** -> **Users** -> **Create User**.
   - Bật Console Access, gán chính sách `AdministratorAccess`.
   - Đăng xuất Root và luôn đăng nhập bằng IAM User qua đường dẫn Console riêng.

---

### Giai đoạn 2: Hạ tầng tính toán EC2 & IP tĩnh Elastic IP
1. **Khởi tạo EC2**:
   - Region: Chọn **Singapore (`ap-southeast-1`)** để độ trễ thấp nhất.
   - OS: **Ubuntu Server 24.04 LTS**.
   - Instance Type: **`t3.micro`** hoặc **`t2.micro`** (Free Tier).
   - Key Pair: Tạo mới `vira-key.pem` (RSA) và tải về máy tính.
   - Firewall (Security Group): Bật **SSH (22)**, **HTTP (80)**, **HTTPS (443)**.
   - Ổ cứng (Storage): Tăng từ 8GB lên **20GB gp3** (Free Tier hỗ trợ tối đa 30GB).
2. **Gán Elastic IP (IP tĩnh)**:
   - Vào **EC2** -> **Elastic IPs** -> **Allocate Elastic IP address**.
   - Chọn IP -> **Actions** -> **Associate Elastic IP address** -> Chọn Instance vừa tạo.
   - *Mục đích*: Cố định IP máy chủ vĩnh viễn (IP mẫu: `18.139.187.188`).

---

### Giai đoạn 3: Phân giải tên miền (DNS Resolution)
1. Trên bảng điều khiển DNS của nhà cung cấp domain (Digitalplat):
   - Bật **DNS Management**.
   - Thêm bản ghi cho Frontend:
     - Type: `A`, Name: `vira`, Value: `18.139.187.188` (-> tạo domain `vira.daoductam.dpdns.org`).
   - Thêm bản ghi cho Backend API:
     - Type: `A`, Name: `api-vira`, Value: `18.139.187.188` (-> tạo domain `api-vira.daoductam.dpdns.org`).

---

### Giai đoạn 4: Kết nối SSH & Tối ưu Linux OS
1. **Sửa quyền file khóa trên Windows**:
   SSH yêu cầu file `.pem` có quyền đọc tối thiểu:
   ```cmd
   icacls D:\vira-key.pem /grant:r <Username_May_Tinh>:R
   ```
2. **Đăng nhập SSH**:
   ```cmd
   ssh -i D:\vira-key.pem ubuntu@18.139.187.188
   ```
3. **Kích hoạt 2GB Swap Memory (Tránh lỗi OOM Killer cho Java)**:
   ```bash
   sudo fallocate -l 2G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   free -h
   ```
4. **Cài đặt Docker Engine & Docker Compose**:
   Cài đặt Docker chính thức từ repository của Docker và thêm user vào nhóm docker (`sudo usermod -aG docker $USER`).

---

### Giai đoạn 5: Reverse Proxy Caddy & Cấp chứng chỉ SSL HTTPS
1. **Tạo cấu trúc thư mục trên server**:
   ```bash
   mkdir -p ~/vira/caddy_data ~/vira/caddy_config ~/vira/uploads ~/vira/mysql_data
   cd ~/vira
   ```
2. **Tạo cấu hình `Caddyfile`**:
   ```caddy
   api-vira.daoductam.dpdns.org {
       reverse_proxy vira-api:8080
   }

   vira.daoductam.dpdns.org {
       root * /var/www/html
       file_server
       try_files {path} /index.html
   }
   ```
   *Caddy sẽ tự động bắt tay với Let's Encrypt để xin và gia hạn SSL cho 2 domain.*

3. **Tạo file biến môi trường bí mật `.env`**:
   Lưu trữ mật khẩu database, JWT Secret riêng biệt trên server.

---

### Giai đoạn 6: Docker Compose Production & Ứng dụng Backend
1. **Tạo `docker-compose.prod.yml`**:
   Kết nối 4 container:
   - `caddy`: Mở port 80/443, mount thư mục `./dist` vào `/var/www/html`.
   - `mysql`: Database MySQL 8.4, mount persistent volume vào `./mysql_data`.
   - `redis`: Redis 7.4 cache và rate limiter.
   - `vira-api`: Spring Boot, liên kết qua mạng nội bộ của Docker.

2. **Build và Triển khai**:
   - Build file JAR Spring Boot: `./mvnw clean package -DskipTests`.
   - Build thư mục Frontend với API URL: `VITE_API_URL=https://api-vira.daoductam.dpdns.org/api/v1 npm run build`.
   - Đẩy file lên server qua `scp`, giải nén và chạy:
     ```bash
     docker compose -f docker-compose.prod.yml up -d
     ```

---

### Giai đoạn 7: Tự động hóa CI/CD với GitHub Actions
File `.github/workflows/deploy.yml` tự động kích hoạt khi có commit push lên nhánh `main`:
1. Checkout mã nguồn.
2. Thiết lập JDK 21 & Node 20, build file JAR và thư mục Frontend `dist/`.
3. Tự động kết nối SSH vào EC2 thông qua GitHub Secrets (`EC2_HOST`, `EC2_SSH_KEY`).
4. Giải nén và reload Docker container mà không gây gián đoạn dịch vụ.

---

## 4. CÁC LỖI KINH ĐIỂN THƯỜNG GẶP & CÁCH KHẮC PHỤC

### 1. `Load key vira-key.pem: Bad permissions` (Khi SSH từ Windows)
- **Nguyên nhân**: File `.pem` bị Windows cấp quyền cho quá nhiều user.
- **Khắc phục**: Dùng `icacls D:\vira-key.pem /grant:r %USERNAME%:R` hoặc reset inheritance.

### 2. `Out of Memory (OOM Killer)` làm tắt Spring Boot / MySQL
- **Nguyên nhân**: Máy chủ `t3.micro` chỉ có 1GB RAM vật lý, Java + MySQL dễ vượt quá dung lượng.
- **Khắc phục**: Luôn tạo 2GB Swap Memory ngay sau khi cài đặt Ubuntu.

### 3. F5 trang React Router bị lỗi 404 / 403
- **Nguyên nhân**: Single Page Application (SPA) chỉ có duy nhất 1 file `index.html`. Khi người dùng gõ trực tiếp URL con (như `/projects`), web server tìm file `/projects` không thấy.
- **Khắc phục**: Thêm lệnh `try_files {path} /index.html` trong Caddy (hoặc custom error page 200 trong CloudFront).

### 4. CORS Error khi gọi API từ Frontend
- **Nguyên nhân**: Domain Frontend (`https://vira.daoductam.dpdns.org`) khác domain Backend (`https://api-vira.daoductam.dpdns.org`), trình duyệt chặn lại nếu Backend không cấp phép.
- **Khắc phục**: Thêm domain Frontend vào `allowedOrigins` trong cấu hình Spring Security CORS của Backend.

### 5. CloudFront bị lỗi `Your account must be verified`
- **Nguyên nhân**: Tài khoản AWS mới hoặc cá nhân bị hạn chế tạm thời dịch vụ CloudFront để chống spam/abuse.
- **Khắc phục**: Gửi Support Case miễn phí trên AWS Console để yêu cầu kích hoạt, hoặc dùng Caddy Server trên EC2 phục vụ static files với hiệu năng tương đương.

---

## 5. CHECKLIST CHO CÁC DỰ ÁN TIẾP THEO

Khi bắt đầu một dự án mới muốn deploy lên AWS, hãy làm theo đúng danh sách này:
- [ ] 1. Bật **AWS Budgets** cảnh báo chi phí trước tiên.
- [ ] 2. Tạo máy chủ **EC2 Ubuntu** ở Region gần nhất (`Singapore`).
- [ ] 3. Gán **Elastic IP** để cố định địa chỉ mạng.
- [ ] 4. Mở Security Group: Port **22**, **80**, **443**.
- [ ] 5. Trỏ bản ghi DNS **A Record** trên trang quản lý domain về Elastic IP.
- [ ] 6. SSH vào server -> Tạo **2GB Swap RAM** -> Cài đặt **Docker**.
- [ ] 7. Cấu hình **Caddy** làm Reverse Proxy tự động cấp phát SSL.
- [ ] 8. Khởi chạy bằng **Docker Compose** và lưu trữ dữ liệu an toàn trên persistent volume.
- [ ] 9. Thiết lập **GitHub Actions Secrets** để tự động hóa toàn bộ quy trình triển khai.
