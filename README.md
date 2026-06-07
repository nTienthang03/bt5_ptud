Dưới đây là bản **trình bày chính thức các bước thực hiện BT5** theo đúng những gì bạn đã làm. Bạn có thể copy vào báo cáo hoặc Git.

---

# Các bước thực hiện bài tập 5

## App Monitor + Alert Data Realtime sử dụng Docker Compose

## 1. Chuẩn bị môi trường

Sử dụng máy ảo Ubuntu làm máy chủ triển khai ứng dụng. Từ máy Windows thật, kết nối SSH vào Ubuntu bằng lệnh:

```bash
ssh thang@192.168.102.97
```

Sau khi đăng nhập thành công, kiểm tra Docker và Docker Compose đã hoạt động:

```bash
docker ps
docker compose version
```

Tạo thư mục project bài tập 5:

```bash
mkdir -p ~/bt5-monitor-app
cd ~/bt5-monitor-app
```

Tạo các thư mục thành phần:

```bash
mkdir api web nginx mariadb
```

Cấu trúc project:

```text
bt5-monitor-app/
│
├── docker-compose.yml
│
├── api/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
├── web/
│   ├── index.html
│   ├── style.css
│   └── app.js
│
├── nginx/
│   └── default.conf
│
└── mariadb/
    └── init.sql
```

---

## 2. Xây dựng hệ thống bằng Docker Compose

## Các container chính trong hệ thống BT5

Sau khi chạy lệnh:

```bash
docker compose up -d --build
```

hệ thống tạo và chạy các container chính sau:

```text
bt5-mariadb
bt5-influxdb
bt5-grafana
bt5-nodered
bt5-flask-api
bt5-nginx
```

### Ý nghĩa từng container

### 1. Container `bt5-mariadb`

```text
bt5-mariadb
```

Container này chạy hệ quản trị cơ sở dữ liệu MariaDB. Trong bài tập, MariaDB được dùng để lưu dữ liệu thời tiết tức thời, tức là dữ liệu mới nhất mà Node-RED lấy được từ API thời tiết.

Dữ liệu được lưu vào database:

```text
monitor_db
```

và bảng:

```text
current_weather
```

Bảng này lưu các thông tin:

```text
location_name   : khu vực
temperature     : nhiệt độ
windspeed       : tốc độ gió
weather_time    : thời gian dữ liệu
status          : trạng thái OK / ALERT LOW / ALERT HIGH
created_at      : thời gian ghi dữ liệu vào database
```

Container này được Flask API truy vấn để lấy dữ liệu mới nhất hiển thị lên web.

---

### 2. Container `bt5-influxdb`

```text
bt5-influxdb
```

Container này chạy InfluxDB. Đây là cơ sở dữ liệu time-series, chuyên dùng để lưu dữ liệu thay đổi theo thời gian.

Trong bài tập, InfluxDB dùng để lưu lịch sử dữ liệu thời tiết, gồm:

```text
temperature
windspeed
location
status
```

Dữ liệu được lưu trong:

```text
Organization: tnut
Bucket: weather_bucket
Measurement: weather
```

InfluxDB là nguồn dữ liệu để Grafana vẽ biểu đồ lịch sử nhiệt độ và tốc độ gió.

---

### 3. Container `bt5-grafana`

```text
bt5-grafana
```

Container này chạy Grafana. Grafana được dùng để trực quan hóa dữ liệu lịch sử lấy từ InfluxDB.

Trong bài tập, Grafana tạo dashboard gồm 2 biểu đồ:

```text
Temperature History
Windspeed History
```

Hai biểu đồ này hiển thị sự thay đổi của nhiệt độ và tốc độ gió theo thời gian. Dashboard Grafana được nhúng vào web frontend bằng thẻ iframe.

Địa chỉ truy cập Grafana local:

```text
http://192.168.102.97:3000
```

Địa chỉ public qua Cloudflare:

```text
https://grafana-bt5.nthangi.id.vn
```

---

### 4. Container `bt5-nodered`

```text
bt5-nodered
```

Container này chạy Node-RED. Node-RED là thành phần xử lý luồng dữ liệu chính của hệ thống.

Node-RED thực hiện các nhiệm vụ:

```text
Lấy dữ liệu thời tiết realtime từ Open-Meteo API
Xử lý dữ liệu nhiệt độ và tốc độ gió
Gán trạng thái OK / ALERT LOW / ALERT HIGH
Ghi dữ liệu tức thời vào MariaDB
Ghi dữ liệu lịch sử vào InfluxDB
Gửi cảnh báo Telegram khi dữ liệu vượt ngưỡng
```

Flow chính trong Node-RED:

```text
Inject
→ HTTP Request
→ Xử lý dữ liệu
→ MariaDB / InfluxDB / Telegram Alert
```

Địa chỉ truy cập Node-RED:

```text
http://192.168.102.97:1880
```

---

### 5. Container `bt5-flask-api`

```text
bt5-flask-api
```

Container này chạy ứng dụng Flask API do nhóm tự xây dựng.

Flask API có nhiệm vụ đọc dữ liệu thời tiết mới nhất từ MariaDB và trả về cho frontend dưới dạng JSON.

Endpoint chính:

```text
/api/current
```

Ví dụ dữ liệu API trả về:

```json
{
  "location_name": "Ha Noi",
  "temperature": 34.4,
  "windspeed": 10.9,
  "weather_time": "2026-06-07T10:15",
  "status": "OK",
  "created_at": "Sun, 07 Jun 2026 10:27:20 GMT"
}
```

Địa chỉ API local:

```text
http://192.168.102.97:5000/api/current
```

Frontend gọi API này để tự động cập nhật dữ liệu realtime lên giao diện web.

---

### 6. Container `bt5-nginx`

```text
bt5-nginx
```

Container này chạy Nginx web server. Nginx có nhiệm vụ phục vụ giao diện web frontend gồm các file:

```text
index.html
style.css
app.js
```

Ngoài ra, Nginx còn proxy request API từ frontend sang Flask API.

Ví dụ frontend gọi:

```javascript
fetch("/api/current")
```

Nginx sẽ chuyển tiếp request này đến container Flask API:

```text
http://flask-api:5000/api/current
```

Địa chỉ web local:

```text
http://192.168.102.97:8080
```

Địa chỉ web public qua Cloudflare:

```text
https://bt5.nthangi.id.vn
```

---

## Tổng kết vai trò các container

```text
bt5-mariadb      : lưu dữ liệu thời tiết tức thời
bt5-influxdb     : lưu dữ liệu lịch sử theo thời gian
bt5-grafana      : vẽ biểu đồ lịch sử từ InfluxDB
bt5-nodered      : lấy dữ liệu realtime, xử lý và gửi cảnh báo
bt5-flask-api    : cung cấp API đọc dữ liệu mới nhất từ MariaDB
bt5-nginx        : chạy web frontend và proxy API
```

Các container này cùng nằm trong mạng Docker nội bộ nên có thể gọi nhau bằng tên service như:

```text
mariadb
influxdb
flask-api
```

Nhờ Docker Compose, toàn bộ hệ thống có thể được khởi động bằng một lệnh:

```bash
docker compose up -d --build
```

và kiểm tra bằng:

```bash
docker ps
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/bc236b2e-ecb0-4b5c-bcbe-3c9dac98ce5d" />


## 3. Tạo cơ sở dữ liệu MariaDB

Trong file `mariadb/init.sql`, tạo database:

```sql
CREATE DATABASE IF NOT EXISTS monitor_db;

USE monitor_db;

CREATE TABLE IF NOT EXISTS current_weather (
    id INT AUTO_INCREMENT PRIMARY KEY,
    location_name VARCHAR(100),
    temperature DOUBLE,
    windspeed DOUBLE,
    weather_time VARCHAR(100),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Bảng `current_weather` dùng để lưu dữ liệu thời tiết mới nhất gồm:

```text
Khu vực
Nhiệt độ
Tốc độ gió
Thời gian dữ liệu
Trạng thái OK / ALERT
Thời gian ghi vào database
```

Kiểm tra dữ liệu trong MariaDB:

```bash
docker exec -it bt5-mariadb mariadb -u monitor_user -p
```

Mật khẩu:

```text
monitor_pass
```

Truy vấn:

```sql
USE monitor_db;
SELECT * FROM current_weather ORDER BY id DESC LIMIT 5;
```

---

## 4. Xây dựng Flask API

Trong thư mục `api`, tạo Flask API để đọc dữ liệu mới nhất từ MariaDB.

API chính:

```text
/api/current
```

Khi gọi API, hệ thống trả dữ liệu dạng JSON:

```json
{
  "location_name": "Ha Noi",
  "temperature": 28.5,
  "windspeed": 9.5,
  "weather_time": "2026-06-05T16:30",
  "status": "OK",
  "created_at": "..."
}
```

Kiểm tra API:

```bash
curl http://127.0.0.1:5000/api/current
```

Hoặc mở trên trình duyệt:

```text
http://192.168.102.97:5000/api/current
```

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/463f8f9c-465b-4d0e-8c25-89568e60454d" />

## 5. Xây dựng giao diện web bằng HTML, CSS, JavaScript

Giao diện web được đặt trong thư mục `web`.

Frontend có chức năng:

```text
Hiển thị dữ liệu thời tiết hiện tại
Tự động gọi API mỗi 5 giây
Hiển thị nhiệt độ, tốc độ gió, trạng thái
Nhúng dashboard Grafana bằng iframe
```

File `web/app.js` gọi API:

```javascript
fetch("/api/current")
```

File `nginx/default.conf` cấu hình Nginx chạy web và proxy API:

```nginx
location /api/ {
    proxy_pass http://flask-api:5000/api/;
}
```

Mở giao diện web local:

```text
http://192.168.102.97:8080
```

---


## 6. Cấu hình Node-RED lấy dữ liệu realtime

Mở Node-RED:

```text
http://192.168.102.97:1880
```

Cài thêm các node cần thiết:

```text
node-red-node-mysql
node-red-contrib-influxdb
node-red-contrib-telegrambot
```

Tạo flow chính:

```text
Inject
→ HTTP Request
→ Xử lý dữ liệu
→ Tạo SQL MariaDB
→ MariaDB Monitor
```

Nguồn dữ liệu sử dụng:

```text
Open-Meteo API
```

API thời tiết Hà Nội:

```text
https://api.open-meteo.com/v1/forecast?latitude=21.0285&longitude=105.8542&current_weather=true
```

Node `Xử lý dữ liệu` lấy các thông số:

```text
temperature
windspeed
time
```

Sau đó phân loại trạng thái:

```text
15°C đến 32°C: OK
Dưới 15°C: ALERT LOW
Trên 32°C: ALERT HIGH
```

---

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/087a59d1-8791-416a-9cf3-f6fdd067531b" />


## 7. Lưu dữ liệu tức thời vào MariaDB

Trong Node-RED, tạo node function `Tạo SQL MariaDB` để sinh câu lệnh INSERT:

```sql
INSERT INTO current_weather 
(location_name, temperature, windspeed, weather_time, status)
VALUES (?, ?, ?, ?, ?)
```

Kết nối MariaDB trong Node-RED:

```text
Host: mariadb
Port: 3306
Database: monitor_db
User: monitor_user
Password: monitor_pass
```

Sau khi inject chạy, dữ liệu được lưu vào bảng `current_weather`.

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/20eca8c2-bd07-4c3b-9954-818fa0096282" />


## 8. Lưu dữ liệu lịch sử vào InfluxDB

Từ node `Xử lý dữ liệu`, tạo thêm nhánh:

```text
Xử lý dữ liệu
→ Tạo dữ liệu InfluxDB
→ InfluxDB Weather
```

Cấu hình InfluxDB:

```text
Version: 2.0
URL: http://influxdb:8086
Token: my-super-token
Organization: tnut
Bucket: weather_bucket
Measurement: weather
```

Dữ liệu được lưu gồm:

```text
temperature
windspeed
location
status
```

InfluxDB dùng để lưu dữ liệu lịch sử theo thời gian.

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/10f01c8c-f4c4-49b8-b454-9a73214f6070" />

## 9. Trực quan hóa dữ liệu bằng Grafana

Mở Grafana:

```text
http://192.168.102.97:3000
```

Đăng nhập:

```text
Username: admin
Password: admin
```

Thêm Data Source InfluxDB:

```text
Query language: Flux
URL: http://influxdb:8086
Organization: tnut
Token: my-super-token
Default bucket: weather_bucket
```

Tạo dashboard:

```text
BT5 Weather Monitor
```

Tạo biểu đồ nhiệt độ với query:

```flux
from(bucket: "weather_bucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "weather")
  |> filter(fn: (r) => r._field == "temperature")
```

Tạo biểu đồ tốc độ gió với query:

```flux
from(bucket: "weather_bucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "weather")
  |> filter(fn: (r) => r._field == "windspeed")
```

Kết quả tạo được 2 biểu đồ:

```text
Temperature History
Windspeed History
```

---

## 10. Nhúng Grafana vào web bằng iframe

Sau khi tạo dashboard Grafana, lấy link dashboard và nhúng vào `web/index.html`:

```html
<iframe
    src="https://grafana-bt5.nthangi.id.vn/d/adwttr4/new-dashboard?orgId=1&from=now-6h&to=now&timezone=browser&refresh=5s&kiosk"
    width="100%"
    height="600"
    frameborder="0">
</iframe>
```

Restart Nginx:

```bash
docker restart bt5-nginx
```

Kết quả: giao diện web hiển thị được cả dữ liệu realtime và biểu đồ lịch sử Grafana.

---

## 11. Cấu hình Telegram Alert

Tạo bot Telegram bằng `@BotFather`.

Thêm bot vào group Telegram.

Lấy Chat ID group bằng API Telegram:

```text
https://api.telegram.org/bot<TOKEN>/getUpdates
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b9570772-72ce-4f04-857f-6224bb637297" />

Chat ID group đã lấy được:

```text
-5260564917
```

Trong Node-RED, tạo nhánh alert:

```text
Xử lý dữ liệu
→ Kiểm tra Alert
→ Tạo nội dung Telegram
→ Telegram sender
```

Node `Kiểm tra Alert` cấu hình:

```text
Property: msg.payload.status
Rule: != OK
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f198438b-45a4-4e13-8e87-dbd63a6d27e8" />

Nội dung cảnh báo Telegram:

```text
[BT5 ALERT WEATHER]
Khu vực: Ha Noi
Trạng thái: ALERT LOW / ALERT HIGH
Nhiệt độ hiện tại: ...
Tốc độ gió: ...
Ngưỡng cho phép: 15°C - 35°C
Thời gian dữ liệu: ...
```

Khi dữ liệu vượt ngưỡng, bot gửi tin nhắn cảnh báo vào group Telegram.

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a40d6a9a-0c02-4993-8d30-c22accb1f734" />


## 12. Public web bằng Cloudflare Tunnel

Do trước đó đã có tunnel `blog-tunnel` chạy healthy, sử dụng luôn tunnel này để thêm route cho BT5.

Thêm 2 route mới:

```text
bt5.nthangi.id.vn
→ http://192.168.102.97:8080
```

```text
grafana-bt5.nthangi.id.vn
→ http://192.168.102.97:3000
```

Do `cloudflared` chạy bằng container `bt4-cloudflared`, không dùng `127.0.0.1`, mà dùng IP máy Ubuntu:

```text
192.168.102.97
```

Restart Cloudflare container:

```bash
docker restart bt4-cloudflared
```

Truy cập web public:

```text
https://bt5.nthangi.id.vn
```

Truy cập Grafana public:

```text
https://grafana-bt5.nthangi.id.vn
```

Kết quả: app BT5 chạy public qua Cloudflare, không ảnh hưởng BT4 vì dùng hostname riêng.

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/130bfb40-22f0-4d4c-a1db-4ba82b955575" />


## 13. Các lệnh chạy toàn bộ bài 5

Mỗi lần muốn chạy lại bài:

```bash
cd ~/bt5-monitor-app
docker compose up -d
docker restart bt4-cloudflared
docker ps
```

Restart toàn bộ container BT5:

```bash
docker restart bt5-nginx bt5-flask-api bt5-mariadb bt5-influxdb bt5-grafana bt5-nodered
```

Kiểm tra API:

```bash
curl http://127.0.0.1:5000/api/current
```

Kiểm tra web:

```bash
curl -I http://127.0.0.1:8080
```

---

## 14. Xuất image và backup bài 5

Tạo thư mục backup:

```bash
cd ~/bt5-monitor-app
mkdir -p backup
```

Lưu các image Docker:

```bash
docker save -o backup/bt5-images.tar \
nginx:alpine \
nodered/node-red:latest \
grafana/grafana:latest \
influxdb:2.7 \
mariadb:10.6 \
bt5-flask-api:latest
```

Nén source code và image:

```bash
tar -czvf bt5-monitor-backup.tar.gz docker-compose.yml api web nginx mariadb backup
```

Kiểm tra file backup:

```bash
ls -lh bt5-monitor-backup.tar.gz
```

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/858d6eb9-6dcf-4075-a6f6-294605f6b73a" />

## 15. Xoá container và khôi phục

Dừng và xoá container BT5:

```bash
cd ~/bt5-monitor-app
docker compose down
```

Load lại image:

```bash
docker load -i backup/bt5-images.tar
```

Khôi phục hệ thống:

```bash
docker compose up -d
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/dd06e7ea-d875-4ec2-9e2b-95f722a5f5e6" />

Kiểm tra:

```bash
docker ps
```

Mở lại:

```text
https://bt5.nthangi.id.vn
```

Nếu web vẫn chạy, việc backup và khôi phục thành công.

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/bb799ef7-4445-48e2-8a0f-0f93bcfac84e" />


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ac40490c-27cd-40de-9ee7-a9bb8ad168b2" />

# Kết luận

Bài tập 5 đã xây dựng thành công hệ thống **App Monitor + Alert Data Realtime** bằng Docker Compose. Hệ thống gồm nhiều service phối hợp với nhau: Node-RED, MariaDB, InfluxDB, Grafana, Flask API và Nginx. Node-RED lấy dữ liệu thời tiết realtime từ Open-Meteo, lưu dữ liệu tức thời vào MariaDB, lưu lịch sử vào InfluxDB, Grafana vẽ biểu đồ, web frontend hiển thị dữ liệu realtime và nhúng biểu đồ lịch sử. Khi dữ liệu vượt ngưỡng, Telegram bot gửi cảnh báo vào group. Ứng dụng cũng được public qua Cloudflare Tunnel bằng domain `bt5.nthangi.id.vn`.
