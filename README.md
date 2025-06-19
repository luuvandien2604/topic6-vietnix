# 1. Xây dựng mô hình reverse proxy kết hợp nginx và apache

## Bước 1: Cài đặt các gói cần thiết
```
sudo apt update && sudo apt install apache2 php libapache2-mod-php php-mysql mysql-server phpmyadmin unzip -y
```

## Bước 2: Apache listen cổng 8080

Mở file:
```
sudo nano /etc/apache2/ports.conf
```
Đảm bảo đang lắng nghe 2 cổng
```
Listen 8080
Listen 8443
```

Sau đó chỉnh virtual host:
```
sudo nano /etc/apache2/sites-available/vdien.laravel.vietnix.tech.conf
sudo nano /etc/apache2/sites-available/vdien.laravel.vietnix.tech.conf
```
Sửa nội dung thành:
```
<VirtualHost *:8080>
    ServerName vdien.laravel.vietnix.tech
    DocumentRoot /var/www/laravel/public

    <Directory /var/www/laravel/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/laravel_http_error.log
    CustomLog ${APACHE_LOG_DIR}/laravel_http_access.log combined
</VirtualHost>

<VirtualHost *:8443>
    ServerName vdien.laravel.vietnix.tech
    DocumentRoot /var/www/laravel/public

    SSLEngine on
    SSLCertificateFile /etc/ssl/zerossl/laravel/certificate.crt
    SSLCertificateKeyFile /etc/ssl/zerossl/laravel/private.key
    SSLCertificateChainFile /etc/ssl/zerossl/laravel/ca_bundle.crt

    <Directory /var/www/laravel/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/laravel_https_error.log
    CustomLog ${APACHE_LOG_DIR}/laravel_https_access.log combined
</VirtualHost>
```
Sau khi sửa nội dung xong tiến hành bật site và reload apache
```
sudo a2ensite vdien.laravel.vietnix.tech.conf
sudo a2ensite vdien.wp.vietnix.tech.conf
sudo systemctl reload apache2
```
## Bước 3: Cấu hình phpMyAdmin trên Apache cổng 8081

Tạo VirtualHost:
```
sudo nano /etc/apache2/sites-available/phpmyadmin.conf
```
Thêm nội dung:
```
<VirtualHost *:8081>
    ServerName phpmyadmin.local
    DocumentRoot /usr/share/phpmyadmin
    <Directory /usr/share/phpmyadmin>
        Options FollowSymLinks
        DirectoryIndex index.php
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Kích hoạt site phpadmin và load lại Apache:
```
sudo ln -s /etc/apache2/sites-available/phpmyadmin.conf /etc/apache2/sites-enabled/
sudo systemctl reload apache2
```
## Bước 4: Cấu hình NGINX làm reverse proxy

Chỉnh sửa file NGINX site tại:
```
nano /etc/nginx/sites-available/vdien.laravel.vietnix.tech
nano /etc/nginx/sites-available/vdien.wp.vietnix.tech
```
Nội dung:
```
server {
    listen 80;
    server_name vdien.laravel.vietnix.tech;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    server_name vdien.laravel.vietnix.tech;

    ssl_certificate /etc/ssl/zerossl/laravel/certificate.crt;
    ssl_certificate_key /etc/ssl/zerossl/laravel/private.key;
    ssl_trusted_certificate /etc/ssl/zerossl/laravel/ca_bundle.crt;

    location / {
        proxy_pass https://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
**Làm tương tự với web Wordpress**

Kích hoạt:
```
sudo ln -s /etc/nginx/sites-available/vdien.laravel.vietnix.tech /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/vdien.wp.vietnix.tech /etc/nginx/sites-enabled/
```
## Bước 5: Khởi động lại dịch vụ
```
sudo systemctl restart apache2
sudo systemctl restart nginx
```
## Bước 6: Để Apache xử lý HTTPS cần bật module SSL:
```
sudo a2enmod ssl
```

# 2. vì sao nginx đứng trước apache 

NGINX có thế mạnh ở khả năng xử lý các request tĩnh như hình ảnh, CSS, JavaScript rất nhanh nhờ kiến trúc non-blocking và tiêu tốn ít tài nguyên hệ thống. Khi đứng ở vai trò reverse proxy, NGINX còn có khả năng điều phối lưu lượng truy cập đến nhiều backend khác nhau, lọc request độc hại, chặn bot, và xử lý SSL hiệu quả. Nhờ đó, NGINX sẽ nhận toàn bộ request từ client, xử lý các phần đơn giản, và chỉ chuyển các request động (PHP, Laravel, WordPress...) cho Apache.

Apache ngược lại, xử lý các nội dung động rất tốt, đặc biệt là các ứng dụng PHP như Laravel và WordPress. Apache hỗ trợ .htaccess, cho phép tùy chỉnh theo thư mục – một tính năng quan trọng mà nhiều framework hoặc CMS hiện nay sử dụng. Ngoài ra, hệ sinh thái module lâu đời của Apache rất phong phú và vẫn được nhiều hệ thống kế thừa.
