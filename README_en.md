# LCOJ Docker

**English** | [Tiếng Việt](README.md)

This repository contains Docker files to run [LCOJ](https://github.com/luyencode/lcoj-docker).

Based on [dmoj-docker](https://github.com/Ninjaclasher/dmoj-docker).

## Requirements

You need to install [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) before getting started.

## Installation

### Step 1: Clone the repository

```sh
git clone --recursive https://github.com/luyencode/lcoj-docker.git
cd lcoj-docker/dmoj
```

From now on, all commands should be run in the `dmoj` directory.

### Step 2: Initialize

Run the initialization script to create necessary directories:

```sh
./scripts/initialize
```

### Step 3: Configuration

Create configuration files from examples:

```sh
cp environment/mysql-admin.env.example environment/mysql-admin.env
cp environment/mysql.env.example environment/mysql.env
cp environment/site.env.example environment/site.env
```

Then edit these files:
- Set MySQL passwords in `mysql.env` and `mysql-admin.env`
- Set host and secret key in `site.env`
- Configure `server_name` in `dmoj/nginx/conf.d/nginx.conf`

### Step 4: Build Docker images

```sh
docker compose build
```

### Step 5: Start services

```sh
docker compose up -d site db redis celery
```

### Step 6: Create database schema

```sh
./scripts/migrate
```

### Step 7: Generate static files

```sh
./scripts/copy_static
```

### Step 8: Load initial data

```sh
./scripts/manage.py loaddata navbar
./scripts/manage.py loaddata language_small
./scripts/manage.py loaddata demo
```

**Note:** The commands above create an admin account with both username and password set to `admin`. You should change the password or remove this account.

Or create your own superuser account:

```sh
./scripts/manage.py createsuperuser
```

## Usage

Start all services:

```sh
docker compose up -d
```

Stop all services:

```sh
docker compose down
```

## Important Notes

### Judge Server

The judge server is not included in this Docker setup. Please refer to [Setting up a Judge](https://vnoi-admin.github.io/vnoj-docs/#/judge/setting_up_a_judge).

The bridge instance is included and will run automatically when you start the services.

### Database Migration

When updating the system, you may need to run migrations:

```sh
./scripts/migrate
```

### Updating Static Files

If static files change, you need to rebuild them:

```sh
./scripts/copy_static
```

### Updating Source Code

Depending on what changed, you need to rebuild or restart:

**If dependencies changed:**

```sh
docker compose up -d --build base site celery bridged wsevent
```

**If only code changed:**

```sh
docker compose restart site celery bridged wsevent
```

### Multiple Nginx Instances

If you already have Nginx running on your host machine, you can change the port in `docker-compose.yml` and configure a proxy pass.

Example Nginx configuration on host machine:

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

In this case, you need to change the port in the Docker container to `10080`.

### Load Balancing

By default, all services run on the same machine. To handle more users, you can distribute services across multiple servers:

- **Central server:** nginx, db, redis, bridged, wsevent
- **Workers:** nginx, site, celery

#### Central Server Setup

Edit `dmoj/nginx/conf.d/nginx.conf` to distribute traffic to workers. Refer to [Nginx docs](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/).

Example configuration:

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

Open these ports in `dmoj/docker-compose.yml`:
- db: 3306
- redis: 6379
- bridged: 9998, 9999
- wsevent: 15100, 15101, 15102

Start services:

```sh
docker compose up -d nginx db redis bridged wsevent
```

#### Worker Setup

Edit `dmoj/environment/site.env` and `dmoj/environment/mysql.env` to point to the central server.

Start services:

```sh
docker compose up -d nginx site celery
```

## Support

If you encounter any issues, please create an issue at [GitHub Issues](https://github.com/luyencode/lcoj-docker/issues).
