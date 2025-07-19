# Laravel Blog Deployment using Docker Compose & Nginx Reverse Proxy

## Server Setup (VM)
### Creating the VM in VirtualBox
- **VM Name**: `devops-vm`
- **OS Type**: Linux / Ubuntu (64-bit)
- **Resources**:
  - RAM: 2048 MB
  - CPU: 2 Cores
  - Storage: 25 GB (Dynamic allocation)
- **Network**: NAT (Intel PRO/1000 MT Desktop)
- **Boot Order**: Optical → Hard Disk
- **Acceleration**: Nested Paging + KVM

### Installation Steps
1. Mount Ubuntu Server 24.04 LTS ISO
2. Create virtual hard disk (VHD) with dynamic allocation
3. Start VM and follow Ubuntu Server installation wizard
4. Configure network settings during installation

## Linux System Hardening & Optimization
### Operating System Hardening
> sudo apt update && sudo apt upgrade -y
> sudo apt install unattended-upgrades
> sudo dpkg-reconfigure unattended-upgrades
> sudo aa-status  # Verify AppArmor

## SSH Security
> sudo nano /etc/ssh/sshd_config
Change: 
[ - Port 2222
- PermitRootLogin no ]
> sudo ufw allow 2222
> sudo systemctl restart sshd

## User Account Hardening
Remove unused users:
sudo deluser --remove-home <username>

## Password policies:
sudo passwd <username>

## MFA Setup:
> sudo apt install libpam-google-authenticator
> google-authenticator  # Run as user
> sudo nano /etc/pam.d/sshd

## Add: auth required pam_google_authenticator.so
Kernel & Performance Tuning
> sudo nano /etc/sysctl.conf

### Add:
[ - net.ipv4.tcp_syncookies=1
- vm.swappiness=10
- kernel.kptr_restrict=2 ]
> sudo sysctl -p

### Firewall Configuration:
> sudo ufw allow OpenSSH
> sudo ufw allow 80/tcp
> sudo ufw allow 443/tcp
> sudo ufw enable


## Nginx Hardening & Optimization
TLS Configuration (self-signed)
> sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt

## Security Headers (Add to Nginx config)
nginx
add_header Strict-Transport-Security "max-age=63072000" always;
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
server_tokens off;
Rate Limiting
nginx
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

location / {
    limit_req zone=one burst=20 nodelay;
    ...
}

Dockerized Laravel Setup
Project Structure
text
├── docker-compose.yml
├── nginx/
│   └── default.conf
├── src/           # Laravel application
└── Dockerfile
Dockerfile
dockerfile
FROM php:8.2-fpm

RUN apt-get update && apt-get install -y \
    git \
    zip \
    unzip \
    libpng-dev \
    libonig-dev \
    libxml2-dev

RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

RUN curl -sS https://getcomposer.org/installer | php -- \
    --install-dir=/usr/local/bin --filename=composer

WORKDIR /var/www
COPY src/ /var/www
RUN composer install
docker-compose.yml
yaml
version: '3.8'

services:
  app:
    build: .
    volumes:
      - ./src:/var/www
    environment:
      - DB_HOST=database
      - DB_DATABASE=laravel
      - DB_USERNAME=root
      - DB_PASSWORD=secret

  webserver:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./src:/var/www
      - ./nginx:/etc/nginx/conf.d

  database:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: laravel
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
Nginx Reverse Proxy Config (nginx/default.conf)
nginx
server {
    listen 80;
    listen 443 ssl;
    
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
    
    server_name _;
    root /var/www/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

## Deployment Steps

### Clone Laravel app:
> git clone https://github.com/syelaoktavia21/laravel-blog.git src
> Build containers:
> docker-compose up -d --build

### Run Laravel migrations:
> docker-compose exec app php artisan migrate

### Verify services:
> docker-compose ps
> Verification
> Access https://<server-ip> (ignore SSL warning)

### Check headers using:
[curl -I https://<server-ip>]

## References
1. Ubuntu System Hardening

2. Red Hat Security Hardening

3. TLS Hardening Guide
