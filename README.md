# 🌤️ WEATHER MONITOR & ALERT SYSTEM — REALTIME

> **BT5 — Docker Compose | Node-RED | InfluxDB | MariaDB | Grafana | Flask | Nginx | Telegram Bot**

---

## 📌 MỤC LỤC

1. [Lý thuyết Docker](#1-lý-thuyết-docker)
   - [Docker là gì?](#11-docker-là-gì)
   - [Các keyword trong docker-compose.yml](#12-các-keyword-trong-docker-composeyml)
   - [Ưu điểm khi triển khai app bằng Docker](#13-ưu-điểm-khi-triển-khai-app-bằng-docker)
   - [Triển khai lên máy chủ không có Internet](#14-triển-khai-lên-máy-chủ-không-có-internet)
2. [Tổng quan hệ thống](#2-tổng-quan-hệ-thống)
3. [Kiến trúc hệ thống](#3-kiến-trúc-hệ-thống)
4. [Yêu cầu hệ thống](#4-yêu-cầu-hệ-thống)
5. [Cấu trúc thư mục](#5-cấu-trúc-thư-mục)
6. [Cấu hình chi tiết từng service](#6-cấu-hình-chi-tiết-từng-service)
7. [Hướng dẫn cài đặt & chạy](#7-hướng-dẫn-cài-đặt--chạy)
8. [Cấu hình Node-RED](#8-cấu-hình-node-red)
9. [Cấu hình Grafana](#9-cấu-hình-grafana)
10. [Cấu hình Telegram Alert Bot](#10-cấu-hình-telegram-alert-bot)
11. [Logic cảnh báo bất thường](#11-logic-cảnh-báo-bất-thường)
12. [Triển khai offline lên máy chủ thật](#12-triển-khai-offline-lên-máy-chủ-thật)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. LÝ THUYẾT DOCKER

### 1.1 Docker là gì?

**Docker** là nền tảng mã nguồn mở cho phép đóng gói ứng dụng cùng toàn bộ dependencies (thư viện, cấu hình, môi trường chạy) vào một đơn vị gọi là **container**.

> 📦 Hãy tưởng tượng container như một chiếc hộp vận chuyển chuẩn hoá — bất kể nội dung bên trong là gì, nó luôn chạy đúng ở mọi nơi có Docker.

<!-- [ẢNH: Sơ đồ so sánh VM vs Docker Container] -->

**Các khái niệm cốt lõi:**

| Khái niệm | Mô tả |
|-----------|-------|
| **Image** | Bản thiết kế (blueprint) của container — chỉ đọc, được tạo từ `Dockerfile` |
| **Container** | Một instance đang chạy của image — có thể start, stop, delete |
| **Dockerfile** | File script định nghĩa cách build một image |
| **Registry** | Kho lưu trữ image — Docker Hub là registry công khai phổ biến nhất |
| **Volume** | Cơ chế lưu trữ dữ liệu bền vững, tồn tại khi container bị xoá |
| **Network** | Mạng ảo nội bộ kết nối các container với nhau |
| **Docker Compose** | Công cụ định nghĩa và quản lý nhiều container cùng lúc qua file YAML |

---

### 1.2 Các keyword trong docker-compose.yml

File `docker-compose.yml` sử dụng cú pháp YAML để mô tả toàn bộ hệ thống multi-container.

---

#### 🔑 `version`
**Ý nghĩa:** Chỉ định phiên bản schema của Compose file.

```yaml
version: "3.8"
```

---

#### 🔑 `services`
**Ý nghĩa:** Khối cha chứa định nghĩa tất cả các service (container) trong hệ thống.

```yaml
services:
  web:
    image: nginx
  db:
    image: mariadb
```

---

#### 🔑 `image`
**Ý nghĩa:** Chỉ định Docker image sẽ dùng để tạo container (lấy từ Docker Hub hoặc local registry).

```yaml
services:
  grafana:
    image: grafana/grafana:10.0.0
```

---

#### 🔑 `build`
**Ý nghĩa:** Chỉ định đường dẫn chứa `Dockerfile` để build image tuỳ chỉnh thay vì dùng image có sẵn.

```yaml
services:
  api:
    build:
      context: ./flask-api
      dockerfile: Dockerfile
```

---

#### 🔑 `container_name`
**Ý nghĩa:** Đặt tên cụ thể cho container (thay vì tên tự động). Tiện dùng khi `docker exec` hoặc log.

```yaml
services:
  nodered:
    container_name: weather_nodered
```

---

#### 🔑 `ports`
**Ý nghĩa:** Map cổng giữa host và container theo dạng `"HOST:CONTAINER"`.

```yaml
services:
  grafana:
    ports:
      - "3000:3000"   # truy cập http://localhost:3000
```

---

#### 🔑 `environment`
**Ý nghĩa:** Truyền biến môi trường vào container (credentials, config, flags).

```yaml
services:
  mariadb:
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: weather_db
      MYSQL_USER: weather_user
      MYSQL_PASSWORD: weather123
```

---

#### 🔑 `env_file`
**Ý nghĩa:** Load biến môi trường từ file `.env` ngoài thay vì viết thẳng vào compose file (bảo mật hơn).

```yaml
services:
  api:
    env_file:
      - .env
```

---

#### 🔑 `volumes`
**Ý nghĩa:** Mount volume vào container — gồm 2 loại:
- **Named volume:** `volume_name:/path` — Docker quản lý
- **Bind mount:** `./host/path:/container/path` — mount trực tiếp thư mục host

```yaml
services:
  influxdb:
    volumes:
      - influxdb_data:/var/lib/influxdb2       # named volume
      - ./influxdb/config:/etc/influxdb2       # bind mount

volumes:
  influxdb_data:    # khai báo named volume ở đây
```

---

#### 🔑 `networks`
**Ý nghĩa:** Định nghĩa mạng ảo nội bộ để các container giao tiếp với nhau theo tên service.

```yaml
services:
  nodered:
    networks:
      - backend_net
  mariadb:
    networks:
      - backend_net

networks:
  backend_net:
    driver: bridge
```

> 💡 Trong cùng network, container gọi nhau bằng **tên service**, VD: `mariadb:3306`

---

#### 🔑 `depends_on`
**Ý nghĩa:** Xác định thứ tự khởi động — service này chỉ start sau khi service kia đã start.

```yaml
services:
  api:
    depends_on:
      - mariadb
      - influxdb
```

> ⚠️ `depends_on` chỉ đợi container **start**, không đợi service bên trong **sẵn sàng** (dùng healthcheck để chắc chắn hơn).

---

#### 🔑 `healthcheck`
**Ý nghĩa:** Định nghĩa lệnh kiểm tra sức khoẻ của container.

```yaml
services:
  mariadb:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

---

#### 🔑 `restart`
**Ý nghĩa:** Chính sách tự khởi động lại khi container bị crash.

| Giá trị | Ý nghĩa |
|---------|---------|
| `no` | Không tự restart |
| `always` | Luôn restart |
| `on-failure` | Restart khi exit code khác 0 |
| `unless-stopped` | Restart trừ khi bị dừng thủ công |

```yaml
services:
  nodered:
    restart: unless-stopped
```

---

#### 🔑 `command`
**Ý nghĩa:** Ghi đè lệnh mặc định (CMD) được định nghĩa trong Dockerfile.

```yaml
services:
  flask_api:
    command: ["python", "app.py", "--host=0.0.0.0"]
```

---

#### 🔑 `working_dir`
**Ý nghĩa:** Đặt thư mục làm việc bên trong container.

```yaml
services:
  api:
    working_dir: /app
```

---

#### 🔑 `labels`
**Ý nghĩa:** Gắn metadata (nhãn) vào container/image/network — dùng để phân loại, quản lý, hoặc tích hợp tool như Traefik.

```yaml
services:
  nginx:
    labels:
      - "com.example.role=webserver"
      - "com.example.version=1.0"
```

---

#### 🔑 `logging`
**Ý nghĩa:** Cấu hình driver và tuỳ chọn ghi log của container.

```yaml
services:
  nodered:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

#### 🔑 `profiles`
**Ý nghĩa:** Gom service vào nhóm — chỉ khởi động khi profile đó được chỉ định.

```yaml
services:
  debug_tool:
    profiles: ["debug"]
```

```bash
docker compose --profile debug up
```

---

### 1.3 Ưu điểm khi triển khai app bằng Docker

<!-- [ẢNH: Infographic Docker benefits] -->

```
┌─────────────────────────────────────────────────────────────┐
│              ƯU ĐIỂM CỦA DOCKER                            │
├──────────────────────┬──────────────────────────────────────┤
│ ✅ Tính nhất quán    │ "Works on my machine" → chạy đúng   │
│                      │ ở mọi môi trường (dev/staging/prod) │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ Cô lập hoàn toàn  │ Mỗi service chạy độc lập, không     │
│                      │ xung đột dependency với nhau        │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ Triển khai nhanh  │ docker compose up -d → toàn bộ     │
│                      │ stack chạy trong vài giây           │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ Scale dễ dàng     │ docker compose scale api=3 tăng     │
│                      │ số instance ngay lập tức            │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ Version control   │ Image được tag version → rollback   │
│                      │ về phiên bản cũ dễ dàng             │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ Nhẹ hơn VM        │ Container share kernel với host,    │
│                      │ khởi động ms thay vì phút như VM    │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ CI/CD thân thiện  │ Tích hợp pipeline build/test/deploy │
│                      │ tự động với GitHub Actions, Jenkins │
├──────────────────────┼──────────────────────────────────────┤
│ ✅ Môi trường sạch   │ Xoá container là sạch hoàn toàn,    │
│                      │ không rác thải hệ thống             │
└──────────────────────┴──────────────────────────────────────┘
```

---

### 1.4 Triển khai lên máy chủ không có Internet

> **Tình huống:** App chạy OK trên laptop → muốn deploy lên máy chủ thật, máy chủ **không có internet**.

<!-- [ẢNH: Sơ đồ luồng export → transfer → import] -->

#### Các bước thực hiện:

**Bước 1: Trên máy laptop (có internet) — Export images**

```bash
# Lưu tất cả images thành file .tar
docker save -o images_bundle.tar \
  nodered/node-red:latest \
  mariadb:10.11 \
  influxdb:2.7 \
  grafana/grafana:10.0.0 \
  nginx:alpine \
  python:3.11-slim

# Nén lại để giảm kích thước
tar -czf images_bundle.tar.gz images_bundle.tar
```

**Bước 2: Đóng gói toàn bộ source code và config**

```bash
tar -czf weather_app.tar.gz \
  docker-compose.yml \
  .env \
  nginx/ \
  flask-api/ \
  nodered-flows/ \
  grafana/
```

**Bước 3: Chuyển files sang máy chủ (USB / SCP nội bộ)**

```bash
scp images_bundle.tar.gz weather_app.tar.gz user@server-ip:/opt/weather/
```

**Bước 4: Trên máy chủ — Cài Docker (nếu chưa có, offline)**

```bash
# Download offline installer từ: https://download.docker.com/linux/static/stable/
tar xzf docker-25.x.x.tgz
sudo cp docker/* /usr/local/bin/
sudo dockerd &
```

**Bước 5: Import images vào Docker local**

```bash
cd /opt/weather/
gunzip images_bundle.tar.gz
docker load -i images_bundle.tar

# Kiểm tra images đã import thành công
docker images
```

**Bước 6: Giải nén project và chạy**

```bash
tar -xzf weather_app.tar.gz
cd weather_app/
cat .env                     # Kiểm tra cấu hình
docker compose up -d
docker compose ps
docker compose logs -f
```

**Bước 7: Kiểm tra hoạt động**

```bash
curl http://localhost:80        # Nginx frontend
curl http://localhost:1880      # Node-RED
curl http://localhost:3000      # Grafana
curl http://localhost:5000/api/weather  # Flask API
```

---

## 2. TỔNG QUAN HỆ THỐNG

Hệ thống **Weather Monitor & Alert** là ứng dụng giám sát thời tiết realtime với khả năng cảnh báo tự động qua Telegram khi dữ liệu vượt ngưỡng cho phép.

<!-- [ẢNH: Screenshot giao diện tổng quan web app] -->

**Luồng dữ liệu chính:**

```
OpenWeatherMap API
        │
        ▼
   [Node-RED]  ──── polling 60s ────
        │
        ├──────────────────────────▶ [MariaDB]    (giá trị tức thời)
        │
        └──────────────────────────▶ [InfluxDB]   (lịch sử time-series)
                                              │
                                              ▼
                                         [Grafana]   (biểu đồ lịch sử)
        │
        ▼ (kiểm tra ngưỡng)
   [Node-RED]
        │
        └──── nếu bất thường ────▶ [Telegram Bot] ──▶ Group Chat
```

```
Người dùng (Browser)
        │
        ▼
    [Nginx] ──▶ HTML/JS/CSS (frontend)
                    │
                    ├── iframe ──▶ [Grafana]
                    └── AJAX  ──▶ [Flask API] ──▶ [MariaDB]
```

---

## 3. KIẾN TRÚC HỆ THỐNG

<!-- [ẢNH: Sơ đồ kiến trúc hệ thống với các mũi tên kết nối] -->

| Service | Image | Port | Vai trò |
|---------|-------|------|---------|
| **Node-RED** | `nodered/node-red:latest` | 1880 | Thu thập & xử lý dữ liệu |
| **MariaDB** | `mariadb:10.11` | 3306 | Lưu giá trị tức thời |
| **InfluxDB** | `influxdb:2.7` | 8086 | Lưu lịch sử time-series |
| **Grafana** | `grafana/grafana:10.0.0` | 3000 | Trực quan hoá biểu đồ |
| **Flask API** | `python:3.11-slim` (build) | 5000 | REST API trung gian |
| **Nginx** | `nginx:alpine` | 80 | Web server & reverse proxy |

---

## 4. YÊU CẦU HỆ THỐNG

| Thành phần | Phiên bản tối thiểu |
|-----------|---------------------|
| Docker Engine | 24.0+ |
| Docker Compose | 2.20+ |
| RAM | 4 GB (khuyến nghị 8 GB) |
| Disk | 10 GB trống |
| OS | Ubuntu 20.04+ / Windows 10 (WSL2) / macOS 12+ |

**Lấy API Key thời tiết (miễn phí):**

1. Đăng ký tại [https://openweathermap.org](https://openweathermap.org)
2. Vào `My API Keys` → Copy key
3. Free tier: 60 calls/minute — đủ dùng cho project này

---

## 5. CẤU TRÚC THƯ MỤC

```
weather-monitor/
│
├── docker-compose.yml          # Định nghĩa toàn bộ stack
├── .env                        # Biến môi trường (KHÔNG commit git)
├── .env.example                # Template biến môi trường
│
├── nginx/
│   ├── nginx.conf              # Cấu hình Nginx
│   └── html/
│       ├── index.html          # Frontend chính
│       ├── style.css           # Giao diện
│       └── app.js              # Logic AJAX/Socket
│
├── flask-api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py                  # Flask REST API
│
├── nodered/
│   └── flows.json              # Export flows Node-RED
│
├── mariadb/
│   └── init.sql                # Script tạo bảng ban đầu
│
├── influxdb/
│   └── config/                 # Config InfluxDB
│
└── grafana/
    ├── provisioning/
    │   ├── datasources/
    │   │   └── datasources.yml
    │   └── dashboards/
    │       ├── dashboards.yml
    │       └── weather.json
    └── dashboards/
```

---

## 6. CẤU HÌNH CHI TIẾT TỪNG SERVICE

### `docker-compose.yml`

```yaml
version: "3.8"

networks:
  weather_net:
    driver: bridge

volumes:
  mariadb_data:
  influxdb_data:
  influxdb_config:
  grafana_data:
  nodered_data:

services:

  # ── NODE-RED ──────────────────────────────────────────────
  nodered:
    image: nodered/node-red:latest
    container_name: weather_nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    environment:
      - TZ=Asia/Ho_Chi_Minh
    volumes:
      - nodered_data:/data
      - ./nodered/flows.json:/data/flows.json
    networks:
      - weather_net
    depends_on:
      mariadb:
        condition: service_healthy
      influxdb:
        condition: service_healthy

  # ── MARIADB ───────────────────────────────────────────────
  mariadb:
    image: mariadb:10.11
    container_name: weather_mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./mariadb/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - weather_net
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── INFLUXDB ──────────────────────────────────────────────
  influxdb:
    image: influxdb:2.7
    container_name: weather_influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: ${INFLUXDB_USER}
      DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUXDB_PASSWORD}
      DOCKER_INFLUXDB_INIT_ORG: ${INFLUXDB_ORG}
      DOCKER_INFLUXDB_INIT_BUCKET: ${INFLUXDB_BUCKET}
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUXDB_TOKEN}
    volumes:
      - influxdb_data:/var/lib/influxdb2
      - influxdb_config:/etc/influxdb2
    networks:
      - weather_net
    healthcheck:
      test: ["CMD", "influx", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── GRAFANA ───────────────────────────────────────────────
  grafana:
    image: grafana/grafana:10.0.0
    container_name: weather_grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Viewer
      GF_SECURITY_ALLOW_EMBEDDING: "true"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - weather_net
    depends_on:
      - influxdb

  # ── FLASK API ─────────────────────────────────────────────
  flask_api:
    build:
      context: ./flask-api
      dockerfile: Dockerfile
    container_name: weather_flask_api
    restart: unless-stopped
    environment:
      - DB_HOST=mariadb
      - DB_PORT=3306
      - DB_NAME=${MYSQL_DATABASE}
      - DB_USER=${MYSQL_USER}
      - DB_PASSWORD=${MYSQL_PASSWORD}
    networks:
      - weather_net
    depends_on:
      mariadb:
        condition: service_healthy

  # ── NGINX ─────────────────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: weather_nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/html:/usr/share/nginx/html:ro
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - weather_net
    depends_on:
      - flask_api
      - grafana
```

---

### `.env` — Biến môi trường

```env
# MariaDB
MYSQL_ROOT_PASSWORD=rootpassword123
MYSQL_DATABASE=weather_db
MYSQL_USER=weather_user
MYSQL_PASSWORD=weather123

# InfluxDB
INFLUXDB_USER=admin
INFLUXDB_PASSWORD=influxpassword123
INFLUXDB_ORG=weather_org
INFLUXDB_BUCKET=weather_bucket
INFLUXDB_TOKEN=my-super-secret-token-replace-this

# Grafana
GRAFANA_USER=admin
GRAFANA_PASSWORD=grafanapassword123

# OpenWeatherMap
OWM_API_KEY=your_openweathermap_api_key_here
OWM_CITY=Hai Phong
OWM_UNITS=metric

# Telegram Bot
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_CHAT_ID=your_group_chat_id_here

# Alert thresholds
TEMP_MIN=15
TEMP_MAX=38
HUMIDITY_MIN=30
HUMIDITY_MAX=90
WIND_MAX=50
```

---

### `mariadb/init.sql` — Khởi tạo database

```sql
CREATE TABLE IF NOT EXISTS weather_current (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    city        VARCHAR(100)   NOT NULL,
    temperature DECIMAL(5,2),
    feels_like  DECIMAL(5,2),
    humidity    INT,
    pressure    INT,
    wind_speed  DECIMAL(5,2),
    wind_dir    INT,
    weather     VARCHAR(100),
    icon        VARCHAR(20),
    visibility  INT,
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP
                         ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY unique_city (city),
    INDEX idx_updated (updated_at)
);
```

---

### `flask-api/app.py` — REST API

```python
from flask import Flask, jsonify
from flask_cors import CORS
import pymysql, os
from datetime import datetime

app = Flask(__name__)
CORS(app)

def get_db():
    return pymysql.connect(
        host=os.environ['DB_HOST'],
        port=int(os.environ.get('DB_PORT', 3306)),
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD'],
        database=os.environ['DB_NAME'],
        cursorclass=pymysql.cursors.DictCursor
    )

@app.route('/api/weather', methods=['GET'])
def get_current_weather():
    try:
        conn = get_db()
        with conn.cursor() as cursor:
            cursor.execute("""
                SELECT city, temperature, feels_like, humidity,
                       pressure, wind_speed, weather, icon,
                       visibility, updated_at
                FROM weather_current ORDER BY updated_at DESC LIMIT 1
            """)
            data = cursor.fetchone()
        conn.close()
        if data:
            data['updated_at'] = data['updated_at'].strftime('%Y-%m-%d %H:%M:%S')
            return jsonify({"status": "ok", "data": data})
        return jsonify({"status": "no_data"}), 404
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

@app.route('/api/health')
def health():
    return jsonify({"status": "healthy", "time": datetime.now().isoformat()})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

### `nginx/nginx.conf`

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    gzip on;
    gzip_types text/plain text/css application/javascript application/json;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy đến Flask API
    location /api/ {
        proxy_pass http://flask_api:5000/api/;
        proxy_set_header Host $host;
        add_header 'Access-Control-Allow-Origin' '*';
    }

    # Proxy Grafana để embed iframe cùng domain
    location /grafana/ {
        proxy_pass http://grafana:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        rewrite ^/grafana/(.*) /$1 break;
    }
}
```

---

### `nginx/html/app.js` — Logic AJAX auto-refresh

```javascript
const API_URL = '/api/weather';
const REFRESH_INTERVAL = 10000; // 10 giây

const THRESHOLDS = {
    temp:     { min: 15, max: 38 },
    humidity: { min: 30, max: 90 },
    wind:     { max: 50 }
};

async function fetchWeather() {
    try {
        const res = await fetch(API_URL, { headers: { 'Cache-Control': 'no-cache' } });
        const result = await res.json();
        if (result.status === 'ok') updateUI(result.data);
    } catch (err) {
        console.error('Lỗi lấy dữ liệu:', err);
    }
}

function updateUI(data) {
    document.getElementById('temperature').textContent = data.temperature.toFixed(1);
    document.getElementById('humidity').textContent    = data.humidity;
    document.getElementById('wind-speed').textContent  = data.wind_speed.toFixed(1);
    document.getElementById('pressure').textContent    = data.pressure;
    document.getElementById('last-update').textContent = 'Cập nhật: ' + data.updated_at;

    checkAlert('alert-temp',     data.temperature, THRESHOLDS.temp);
    checkAlert('alert-humidity', data.humidity,    THRESHOLDS.humidity);
    checkAlert('alert-wind',     data.wind_speed,  THRESHOLDS.wind);
}

function checkAlert(badgeId, value, threshold) {
    const badge = document.getElementById(badgeId);
    if (!badge) return;
    if (threshold.min !== undefined && value < threshold.min) {
        badge.textContent = '⚠️ THẤP'; badge.className = 'alert-badge alert-low';
    } else if (value > threshold.max) {
        badge.textContent = '🔴 CAO';  badge.className = 'alert-badge alert-high';
    } else {
        badge.textContent = '✅ OK';   badge.className = 'alert-badge alert-ok';
    }
}

fetchWeather();
setInterval(fetchWeather, REFRESH_INTERVAL);
```

---

## 7. HƯỚNG DẪN CÀI ĐẶT & CHẠY

**Bước 1: Clone project**

```bash
git clone https://github.com/yourname/weather-monitor.git
cd weather-monitor
```

**Bước 2: Cấu hình biến môi trường**

```bash
cp .env.example .env
nano .env    # Điền API key, passwords
```

**Bước 3: Khởi động stack**

```bash
docker compose up -d --build
docker compose logs -f
```

**Bước 4: Kiểm tra trạng thái**

```bash
docker compose ps
```

```
NAME                   STATUS          PORTS
weather_nodered        Up (healthy)    0.0.0.0:1880->1880/tcp
weather_mariadb        Up (healthy)    3306/tcp
weather_influxdb       Up (healthy)    0.0.0.0:8086->8086/tcp
weather_grafana        Up (healthy)    0.0.0.0:3000->3000/tcp
weather_flask_api      Up              5000/tcp
weather_nginx          Up              0.0.0.0:80->80/tcp
```

**Bước 5: Truy cập các dịch vụ**

| Dịch vụ | URL | Tài khoản |
|---------|-----|-----------|
| **Web App** | http://localhost | — |
| **Node-RED** | http://localhost:1880 | — |
| **Grafana** | http://localhost:3000 | admin / (xem .env) |
| **InfluxDB UI** | http://localhost:8086 | admin / (xem .env) |
| **API Test** | http://localhost/api/weather | — |

---

## 8. CẤU HÌNH NODE-RED

<!-- [ẢNH: Screenshot Node-RED flow diagram] -->

**Cài node packages cần thiết trong Node-RED:**

```
Manage Palette → Install:
  node-red-node-mysql
  node-red-contrib-influxdb
  node-red-contrib-telegrambot
```

**HTTP Request Node — gọi OpenWeatherMap:**
```
URL: https://api.openweathermap.org/data/2.5/weather
     ?q=Hai Phong,VN&appid={OWM_API_KEY}&units=metric&lang=vi
```

**Function Node — Parse dữ liệu:**

```javascript
const w = JSON.parse(msg.payload);
msg.payload = {
    city:        w.name,
    temperature: w.main.temp,
    feels_like:  w.main.feels_like,
    humidity:    w.main.humidity,
    pressure:    w.main.pressure,
    wind_speed:  w.wind.speed * 3.6,   // m/s → km/h
    wind_dir:    w.wind.deg,
    weather:     w.weather[0].description,
    icon:        w.weather[0].icon,
    visibility:  w.visibility
};
return msg;
```

**MySQL Node — Ghi vào MariaDB:**

```sql
INSERT INTO weather_current
  (city, temperature, feels_like, humidity, pressure,
   wind_speed, wind_dir, weather, icon, visibility)
VALUES
  ('{payload.city}', {payload.temperature}, {payload.feels_like},
   {payload.humidity}, {payload.pressure}, {payload.wind_speed},
   {payload.wind_dir}, '{payload.weather}', '{payload.icon}', {payload.visibility})
ON DUPLICATE KEY UPDATE
  temperature = VALUES(temperature), humidity = VALUES(humidity),
  wind_speed  = VALUES(wind_speed),  weather  = VALUES(weather),
  updated_at  = CURRENT_TIMESTAMP;
```

**Function Node — Format cho InfluxDB:**

```javascript
msg.payload = [{
    measurement: "weather",
    tags: { city: msg.payload.city },
    fields: {
        temperature: msg.payload.temperature,
        humidity:    msg.payload.humidity,
        wind_speed:  msg.payload.wind_speed,
        pressure:    msg.payload.pressure
    }
}];
return msg;
```

---

## 9. CẤU HÌNH GRAFANA

<!-- [ẢNH: Screenshot Grafana dashboard với biểu đồ nhiệt độ] -->

**Auto-provisioning datasource** (`grafana/provisioning/datasources/datasources.yml`):

```yaml
apiVersion: 1
datasources:
  - name: InfluxDB_Weather
    type: influxdb
    access: proxy
    url: http://influxdb:8086
    jsonData:
      version: Flux
      organization: weather_org
      defaultBucket: weather_bucket
    secureJsonData:
      token: my-super-secret-token-replace-this
    isDefault: true
```

**Query Flux — Nhiệt độ 24 giờ:**

```flux
from(bucket: "weather_bucket")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "weather")
  |> filter(fn: (r) => r._field == "temperature")
  |> aggregateWindow(every: 10m, fn: mean, createEmpty: false)
```

**Nhúng Grafana vào web qua iframe:**

```html
<iframe
  src="http://localhost:3000/d/weather-dashboard/weather
       ?orgId=1&refresh=30s&from=now-24h&to=now&kiosk=tv"
  width="100%" height="500" frameborder="0">
</iframe>
```

---

## 10. CẤU HÌNH TELEGRAM ALERT BOT

<!-- [ẢNH: Screenshot tin nhắn alert trong Telegram group] -->

**Tạo Bot:**
1. Nhắn [@BotFather](https://t.me/BotFather) → `/newbot` → nhận **Bot Token**
2. Thêm bot vào group chat (đã có thành viên + thêm `1875746636`)
3. Lấy Chat ID: `https://api.telegram.org/bot{TOKEN}/getUpdates`

**Node-RED — Function Node gửi Telegram khi có cảnh báo:**

```javascript
const THRESHOLDS = {
    temperature: { min: 15, max: 38 },
    humidity:    { min: 30, max: 90 },
    wind_speed:  { max: 50 }
};

const d = msg.payload;
const alerts = [];
const now = new Date().toLocaleString('vi-VN', { timeZone: 'Asia/Ho_Chi_Minh' });

if (d.temperature > THRESHOLDS.temperature.max)
    alerts.push(`🔴 NHIỆT ĐỘ QUÁ CAO: ${d.temperature}°C (max: ${THRESHOLDS.temperature.max}°C)`);
else if (d.temperature < THRESHOLDS.temperature.min)
    alerts.push(`🔵 NHIỆT ĐỘ QUÁ THẤP: ${d.temperature}°C (min: ${THRESHOLDS.temperature.min}°C)`);

if (d.humidity > THRESHOLDS.humidity.max)
    alerts.push(`💧 ĐỘ ẨM QUÁ CAO: ${d.humidity}% (max: ${THRESHOLDS.humidity.max}%)`);
else if (d.humidity < THRESHOLDS.humidity.min)
    alerts.push(`🏜️ ĐỘ ẨM QUÁ THẤP: ${d.humidity}% (min: ${THRESHOLDS.humidity.min}%)`);

if (d.wind_speed > THRESHOLDS.wind_speed.max)
    alerts.push(`💨 GIÓ QUÁ MẠNH: ${d.wind_speed.toFixed(1)} km/h (max: ${THRESHOLDS.wind_speed.max} km/h)`);

if (alerts.length > 0) {
    msg.payload = {
        chatId:    process.env.TELEGRAM_CHAT_ID,
        type:      'message',
        parseMode: 'Markdown',
        content: [
            `⚠️ *CẢNH BÁO THỜI TIẾT BẤT THƯỜNG*`,
            `📍 Địa điểm: *${d.city}*`,
            `🕐 Thời gian: ${now}`,
            ``,
            ...alerts,
            ``,
            `📊 Chi tiết: cảm giác như ${d.feels_like}°C | áp suất ${d.pressure} hPa`
        ].join('\n')
    };
    return msg;
}
return null;  // Không gửi nếu bình thường
```

---

## 11. LOGIC CẢNH BÁO BẤT THƯỜNG

<!-- [ẢNH: Sơ đồ logic kiểm tra ngưỡng alert] -->

```
       Giá trị đo được
               │
       ┌───────▼───────┐
       │  value < MIN? │──YES──▶ 🔵 ALERT LOW  → Telegram
       └───────┬───────┘
               │ NO
       ┌───────▼───────┐
       │  value > MAX? │──YES──▶ 🔴 ALERT HIGH → Telegram
       └───────┬───────┘
               │ NO
            ✅ OK — Không gửi
```

| Thông số | ALERT LOW | Vùng OK | ALERT HIGH |
|----------|-----------|---------|------------|
| Nhiệt độ | < 15°C | 15 – 38°C | > 38°C |
| Độ ẩm | < 30% | 30 – 90% | > 90% |
| Tốc độ gió | — | 0 – 50 km/h | > 50 km/h |
| Áp suất | < 980 hPa | 980 – 1030 hPa | > 1030 hPa |

**Ví dụ tin nhắn Telegram:**

```
⚠️ CẢNH BÁO THỜI TIẾT BẤT THƯỜNG
📍 Địa điểm: Hai Phong
🕐 Thời gian: 14/06/2025, 14:35:20

🔴 NHIỆT ĐỘ QUÁ CAO: 39.2°C (max: 38°C)
💧 ĐỘ ẨM QUÁ CAO: 93% (max: 90%)

📊 Chi tiết: cảm giác như 42.1°C | áp suất 1008 hPa
```

---

## 12. TRIỂN KHAI OFFLINE LÊN MÁY CHỦ THẬT

### Checklist triển khai:

- [ ] Export toàn bộ Docker images: `docker save -o images_bundle.tar [images...]`
- [ ] Nén: `tar -czf images_bundle.tar.gz images_bundle.tar`
- [ ] Đóng gói source: `tar -czf weather_app.tar.gz docker-compose.yml .env nginx/ flask-api/ nodered/ grafana/`
- [ ] Chuyển sang máy chủ qua USB / LAN nội bộ
- [ ] Cài Docker Engine offline nếu cần
- [ ] `docker load -i images_bundle.tar`
- [ ] Chỉnh `.env` cho môi trường production
- [ ] `docker compose up -d`
- [ ] Kiểm tra `docker compose ps` và `docker compose logs`
- [ ] Mở firewall port 80, 3000, 1880 nếu cần truy cập ngoài

---

## 13. TROUBLESHOOTING

**Container không start:**
```bash
docker compose logs [service_name]
docker stats
docker compose restart nodered
```

**MariaDB không kết nối được:**
```bash
docker exec -it weather_nodered sh
ping mariadb   # Test DNS resolution trong network
```

**Grafana không hiển thị trong iframe:**
```bash
# Kiểm tra .env có đủ 2 dòng sau không:
GF_SECURITY_ALLOW_EMBEDDING=true
GF_AUTH_ANONYMOUS_ENABLED=true
```

**Telegram Bot không gửi được:**
```bash
curl -X POST "https://api.telegram.org/bot{TOKEN}/sendMessage" \
  -d "chat_id={CHAT_ID}&text=Test message"
```

**Reset toàn bộ (xoá dữ liệu):**
```bash
docker compose down -v        # Dừng và xoá volumes
docker compose up -d --build  # Chạy lại từ đầu
```

---

## 📎 TÀI LIỆU THAM KHẢO

| Tài liệu | Link |
|----------|------|
| Docker Compose Reference | https://docs.docker.com/compose/compose-file/ |
| Node-RED Documentation | https://nodered.org/docs/ |
| OpenWeatherMap API | https://openweathermap.org/api |
| InfluxDB Flux Query | https://docs.influxdata.com/flux/ |
| Grafana Provisioning | https://grafana.com/docs/grafana/latest/administration/provisioning/ |
| Telegram Bot API | https://core.telegram.org/bots/api |

---

> 📝 **Lưu ý:** Các vị trí `<!-- [ẢNH: ...] -->` là nơi bạn chèn ảnh minh hoạ vào.

---

*README — BT5: Docker Compose | Weather Monitor & Alert Realtime*  
*Tác giả: [Tên sinh viên] — [MSSV]*
