# n8n-self-hosted

<br>

![License](https://img.shields.io/badge/license-MIT-green)

<br>

## YÊU CẦU

1. Máy chủ VPS (Azure, AWS, Google...) cài đặt hệ điều hành Linux (Ubuntu)
2. Domain cá nhân

<br>

## CÁC BƯỚC THỰC HIỆN

### Bước 1:

<br>

SSH vào VPS

<br>

### Bước 2:

<br>

Cập nhật hệ thống và cài đặt Docker các thành phần cần sử dụng

Mở Terminal và gõ:

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo apt-get install docker-compose-plugin
```
<br>

### Bước 3:

<br>

Khởi tạo thư mục dự án

Mở Terminal và gõ:

```bash
mkdir ~/n8n-server
```

Truy cập vào thư mục dự án vừa tạo

```bash
cd ~/n8n-server
```
<br>

### Bước 4:

<br>

Khởi tạo môi trường `.env`

Mở Terminal và gõ:

```bash
sudo nano .env
```
Sao chép nội dung bên dưới vào môi trường `.env` của bạn và lưu lại bằng cách nhấn: `Ctrl + X` nhập `Y` và nhấn `Enter`

```env
# Tên miền của bạn
DOMAIN_NAME=example.com.vn
# Tên miền con (Ví dụ: https://n8n.example.com.vn để truy cập vào N8N)
SUBDOMAIN=n8n
# Email xác thực SSL (Nhập bất kỳ theo ý)
SSL_EMAIL=example@gmail.com
# Timezone
GENERIC_TIMEZONE=Asia/Ho_Chi_Minh
# Tên Database
POSTGRES_DB=n8n
# Tài khoản Database
POSTGRES_USER=n8n
# Mật khẩu Database
POSTGRES_PASSWORD=trgHgfu8xqkl
```

<br>

### Bước 5:

<br>

Khởi tạo tệp tin `docker-compose.yml`

Mở Terminal và gõ:

<br>

```bash
sudo nano docker-compose.yml
```

<br>

Sao chép nội dung bên dưới vào tệp tin `docker-compose.yml` của bạn và lưu lại bằng cách nhấn: `Ctrl + X` nhập `Y` và nhấn `Enter`

```yaml
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  redis:
    image: redis:7-alpine
    restart: always

  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - QUEUE_MODE=redis
      - QUEUE_REDIS_HOST=redis
      - QUEUE_REDIS_PORT=6379
    volumes:
      - n8n_data:/home/node/.n8n
      - /local-files:/files
    depends_on:
      - postgres
      - redis

volumes:
  traefik_data:
  n8n_data:
  postgres_data:
```

<br>

### Bước 6:

<br>

Khởi chạy

Mở terminal và gõ:

```bash
sudo docker compose --env-file .env up -d
```

Kiểm tra các container đang chạy

```bash
sudo docker ps
```

<br>

### Bước 7:

<br>

Bạn tiến hành truy cập vào trang quản trị của Domain của bạn để tiến hành trỏ Domain đã thêm ở `Bước 4` về địa chỉ IP của máy chủ n8n

Tạo bản ghi A và thêm subdomain và nhập địa chỉ IP của máy chủ n8n

<br>

### Bước 8:

<br>

Để truy cập bạn thay `<subdomain>.<domain-cua-ban>` bằng thông tin của bạn ở `Bước 4`

```html
https://<subdomain>.<domain-cua-ban>
```

<br>

Ví dụ:

```html
https://n8n.example.com.vn/
```
<br>
<br>
<br>

## NẾU BẠN KHÔNG CÓ DOMAIN THÌ CÓ THỂ THAM KHẢO CÁCH SAU

<br>

Bạn điều chỉnh lại cấu hình trong `.env`

```env
DOMAIN_NAME=localhost
SUBDOMAIN=n8n
N8N_HOST=127.0.0.1
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://127.0.0.1:5678/
```

<br>

Điều chỉnh lại tệp tin `docker-compose.yml`

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=127.0.0.1
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://127.0.0.1:5678/
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8npassword
      - QUEUE_MODE=redis
      - QUEUE_REDIS_HOST=redis
      - QUEUE_REDIS_PORT=6379
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: n8npassword
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  n8n_data:
  postgres_data:
```

<br>

Sau đó chạy:

```bash
sudo docker compose up -d
```

<br>

Truy cập:

<br>

```text
http://localhost:5678
```

<br>

```text
http://<IP-Server>:5678
```
