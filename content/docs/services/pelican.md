---
title: "Pelican Panel Installation"
date: 2025-06-01
weight: 20
draft: false
---

# Pelican Panel Installation

## Overview

{{< hint info >}}
This guide covers the installation of both Pelican Panel and Wings daemon for game server management. Installation instructions were tested using Debian 12.
{{< /hint >}}

[Pelican](https://pelican.dev) Panel is a free, open source game server control panel with a user friendly interface.

- **Panel**: Web based management interface
- **Wings**: Server daemon for game server management utilizes docker containers

---

## Manual Installation

### Panel Manual Install

#### 1. Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites for adding repositories
sudo apt install -y curl gnupg2 software-properties-common lsb-release ca-certificates apt-transport-https

# Add PHP repository
curl -sSL https://packages.sury.org/php/apt.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/php.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list

# Update package list with new repository
sudo apt update

# Install core dependencies
sudo apt install -y php8.3 php8.3-{cli,common,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip,intl,sqlite3} nginx tar unzip git
```

#### 2. Install and Configure MariaDB

```bash
# Install MariaDB
sudo curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash
sudo apt install -y mariadb-server

# Create database and user
sudo mysql -u root -p
```

In MySQL prompt:
```sql
CREATE USER 'pelican'@'127.0.0.1' IDENTIFIED BY 'somePassword';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'pelican'@'127.0.0.1';
FLUSH PRIVILEGES;
exit;
```

#### 3. Download and Configure Pelican

```bash
# Create directory and download
sudo mkdir -p /var/www/pelican
cd /var/www/pelican
sudo curl -L https://github.com/pelican-dev/panel/releases/latest/download/panel.tar.gz | sudo tar -xzv

# Install Composer
sudo curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer

# Install dependencies
sudo composer install --no-dev --optimize-autoloader

# Set permissions
sudo chmod -R 755 storage/* bootstrap/cache/
sudo chown -R www-data:www-data /var/www/pelican

# Setup environment
sudo php artisan p:environment:setup
```

#### 4. Configure Nginx

Remove default config:
```bash
sudo rm /etc/nginx/sites-enabled/default
```

Create new config:
```bash
sudo nano /etc/nginx/sites-available/pelican.conf
```

Add configuration:
```nginx
server {
    listen 80;
    server_name _;
    root /var/www/pelican/public;
    index index.php;
    access_log /var/log/nginx/pelican.app-access.log;
    error_log /var/log/nginx/pelican.app-error.log error;
    client_max_body_size 100m;
    client_body_timeout 120s;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }
    
    location ~ /\.ht {
        deny all;
    }
}
```

Enable site:
```bash
sudo ln -s /etc/nginx/sites-available/pelican.conf /etc/nginx/sites-enabled/pelican.conf
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl restart php8.3-fpm
```

#### 5. Final Setup

```bash
cd /var/www/pelican
sudo chown -R www-data:www-data storage
sudo chmod -R 755 storage
sudo chown -R www-data:www-data bootstrap/cache
sudo chmod -R 755 bootstrap/cache
```

#### 6. Web Installation

Access: `http://your-server-ip/installer`

Database Settings:
- Type: MySQL
- Host: 127.0.0.1
- Port: 3306
- Database: panel
- Username: pelican
- Password: somePassword

### Wings Manual Install

#### 1. Prerequisites Check

```bash
# Check virtualization type (Should not show OpenVZ or LXC)
sudo systemctl status
systemd-detect-virt

# Install dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates gnupg2 sudo
```

#### 2. Install Docker

```bash
# Add Docker repository
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Enable Docker
sudo systemctl enable --now docker
```

#### 3. Install Wings

```bash
# Download Wings
sudo mkdir -p /etc/pelican
sudo curl -L -o /usr/local/bin/wings "https://github.com/pelican-dev/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
sudo chmod u+x /usr/local/bin/wings
```

#### 4. Configure Wings

Get node configuration from the Panel admin area and save it:
```bash
sudo nano /etc/pelican/config.yml
# Paste configuration here
```

#### 5. Create Wings Service

```bash
sudo nano /etc/systemd/system/wings.service
```

Add this content:
```ini
[Unit]
Description=Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pelican
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

#### 6. Enable and Start Wings

```bash
sudo systemctl enable --now wings
```

#### 7. Storage Setup

```bash
# Create directory for server data
sudo mkdir -p /var/lib/pelican
sudo chmod -R 755 /var/lib/pelican
```

#### 8. Enable Swap Accounting

```bash
sudo nano /etc/default/grub
# Add or modify: GRUB_CMDLINE_LINUX_DEFAULT="swapaccount=1"
sudo update-grub
sudo reboot
```