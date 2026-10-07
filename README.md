# Bài 3: MySQL + Systemd cho Spring Boot (cổng 8082)

## 1. MySQL
```sql
CREATE DATABASE IF NOT EXISTS springboot_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'spring-admin'@'localhost' IDENTIFIED BY 'SpringSecure@123';
GRANT ALL PRIVILEGES ON springboot_db.* TO 'spring-admin'@'localhost';
FLUSH PRIVILEGES;
```
Import: `sudo mysql < ex3.sql`. File SQL: xem `ex3.sql` (lưu cùng repo).
Kết quả:
```
Grants for spring-admin@localhost
GRANT USAGE ON *.* TO `spring-admin`@`localhost`
GRANT ALL PRIVILEGES ON `springboot_db`.* TO `spring-admin`@`localhost`
```
Đăng nhập kiểm chứng: `mysql --defaults-extra-file=my.cnf -e 'SELECT CURRENT_USER();'`
-> `spring-admin@localhost` (mật khẩu SpringSecure@123 đúng).

## 2. User chạy ứng dụng
```bash
sudo useradd -r -s /usr/sbin/nologin spring-runner
```
```
uid=996(spring-runner) gid=986(spring-runner) groups=986(spring-runner)
spring-runner:x:996:986::/home/spring-runner:/usr/sbin/nologin
```
-> system user, shell nologin (không login được).

## 3. Ứng dụng
App thay thế `app.jar` (JDK HttpServer, không cần build ngoài): `SpringApp.java`
biên dịch `javac`, đóng jar, đặt tại `/opt/spring-app/app.jar` (owner spring-runner).
```bash
sudo mkdir -p /opt/spring-app
sudo cp app.jar /opt/spring-app/app.jar
sudo chown -R spring-runner:spring-runner /opt/spring-app
```

## 4. Service tự viết: `/etc/systemd/system/spring-app.service`
Nội dung đầy đủ xem file `spring-app.service` trong repo này.
```bash
sudo cp spring-app.service /etc/systemd/system/spring-app.service
sudo systemctl daemon-reload
sudo systemctl enable --now spring-app.service
```

## 5. Kiểm tra
```
$ sudo systemctl status spring-app.service --no-pager
Active: active (running) since Tue 2026-10-06 09:05:29 UTC; 3s ago
Main PID: 58628 (java)
CGroup: ... /usr/bin/java -jar /opt/spring-app/app.jar
... Listening on 8082

$ sudo ss -tlnp | grep 8082
LISTEN 0 50 *:8082 *:* users:(("java",pid=58628,fd=5))

$ curl -s http://127.0.0.1:8082/api/health
{"status":"UP"}

$ ps -o user,comm -p 58628
USER     COMMAND
spring-+ java
```
-> Dịch vụ running, cổng 8082 do tiến trình java của user spring-runner giữ.
Tự khởi động lại: `Restart=on-failure`, `RestartSec=10`.
