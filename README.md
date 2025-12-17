# LCOJ Docker

[English](README_en.md) | **Tiếng Việt**

Repository này chứa các file Docker để chạy [LCOJ](https://github.com/luyencode/lcoj-docker).

Dựa trên [dmoj-docker](https://github.com/Ninjaclasher/dmoj-docker) và [vnoj-docker](https://github.com/VNOI-Admin/vnoj-docker).

## Yêu cầu

Bạn cần cài đặt [Docker](https://www.docker.com/) và [Docker Compose](https://docs.docker.com/compose/) trước khi bắt đầu.

## Cài đặt

### Bước 1: Tải mã nguồn

```sh
git clone --recursive https://github.com/luyencode/lcoj-docker.git
cd lcoj-docker/dmoj
```

Từ giờ, tất cả các lệnh đều chạy trong thư mục `dmoj`.

### Bước 2: Khởi tạo

Chạy script khởi tạo để tạo các thư mục cần thiết:

```sh
./scripts/initialize
```

### Bước 3: Cấu hình

Tạo các file cấu hình từ file mẫu:

```sh
cp environment/mysql-admin.env.example environment/mysql-admin.env
cp environment/mysql.env.example environment/mysql.env
cp environment/site.env.example environment/site.env
```

Sau đó, chỉnh sửa các file này:
- Đặt mật khẩu MySQL trong `mysql.env` và `mysql-admin.env`
- Đặt host và secret key trong `site.env`
- Cấu hình `server_name` trong `dmoj/nginx/conf.d/nginx.conf`

### Bước 4: Build Docker images

```sh
docker compose build
```

### Bước 5: Khởi động các dịch vụ

```sh
docker compose up -d site db redis celery
```

### Bước 6: Tạo database

```sh
./scripts/migrate
```

### Bước 7: Tạo static files

```sh
./scripts/copy_static
```

### Bước 8: Load dữ liệu mẫu

```sh
./scripts/manage.py loaddata navbar
./scripts/manage.py loaddata language_small
./scripts/manage.py loaddata demo
```

**Lưu ý:** Lệnh trên tạo tài khoản admin với username và password đều là `admin`. Bạn nên đổi mật khẩu hoặc xóa tài khoản này.

Hoặc tạo tài khoản admin của riêng bạn:

```sh
./scripts/manage.py createsuperuser
```

## Sử dụng

Khởi động tất cả dịch vụ:

```sh
docker compose up -d
```

Dừng tất cả dịch vụ:

```sh
docker compose down
```

## Các lưu ý quan trọng

### Judge server

Judge server không có trong Docker setup này. Vui lòng tham khảo [Hướng dẫn cài đặt Judge](https://luyencode.github.io/docs/#/judge/setting_up_a_judge).

Bridge instance đã được bao gồm và sẽ tự động chạy khi bạn khởi động.

### Cập nhật database (Migration)

Khi cập nhật hệ thống, bạn có thể cần chạy migration:

```sh
./scripts/migrate
```

### Cập nhật Static Files

Nếu static files thay đổi, bạn cần rebuild:

```sh
./scripts/copy_static
```

### Cập nhật mã nguồn

Tùy vào phần nào thay đổi, bạn cần rebuild hoặc restart:

**Nếu thay đổi dependencies:**

```sh
docker compose up -d --build base site celery bridged wsevent
```

**Nếu chỉ thay đổi code:**

```sh
docker compose restart site celery bridged wsevent
```

### Sử dụng nhiều Nginx instances

Nếu bạn đã có Nginx chạy trên máy host, bạn có thể đổi port trong `docker-compose.yml` và cấu hình proxy pass.

Ví dụ cấu hình Nginx trên máy host:

```nginx
server {
    listen 80;
    listen [::]:80;

    add_header X-UA-Compatible "IE=Edge,chrome=1";
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    location / {
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://127.0.0.1:10080/;
    }
}
```

Trong trường hợp này, bạn cần đổi port trong Docker container thành `10080`.

### Load balancing (Phân tải)

Mặc định, tất cả dịch vụ chạy trên cùng một máy. Để xử lý nhiều người dùng hơn, bạn có thể phân tán dịch vụ ra nhiều server:

- **Server trung tâm:** nginx, db, redis, bridged, wsevent
- **Các worker:** nginx, site, celery

#### Cấu hình server trung tâm

Chỉnh sửa `dmoj/nginx/conf.d/nginx.conf` để phân phối traffic đến các worker. Tham khảo [Nginx docs](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/).

Ví dụ:

```nginx
upstream site {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}

server {
    listen 80;
    listen [::]:80;

    location / {
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://site/;
    }

    location /event/ {
        proxy_pass http://wsevent:15100/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }

    location /channels/ {
        proxy_read_timeout 120;
        proxy_pass http://wsevent:15102/;
    }
}
```

Mở các port sau trong `dmoj/docker-compose.yml`:
- db: 3306
- redis: 6379
- bridged: 9998, 9999
- wsevent: 15100, 15101, 15102

Khởi động dịch vụ:

```sh
docker compose up -d nginx db redis bridged wsevent
```

#### Cấu hình worker

Chỉnh sửa `dmoj/environment/site.env` và `dmoj/environment/mysql.env` để trỏ đến server trung tâm.

Khởi động dịch vụ:

```sh
docker compose up -d nginx site celery
```

## Tài liệu chi tiết

Xem tài liệu đầy đủ tại: [https://docs.luyencode.net](https://docs.luyencode.net)

## Hỗ trợ

Nếu gặp vấn đề:
- Tạo issue tại [GitHub Issues](https://github.com/luyencode/lcoj-docker/issues)
- Nếu bạn quá mệt mỏi để tự mình cài đặt, LCOJ sẽ hỗ trợ bạn cài đặt miễn phí: [https://luyencode.net/about/#lien-he](https://luyencode.net/about/#lien-he)
