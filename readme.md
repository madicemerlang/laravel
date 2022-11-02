# Laravel dengan nginx mysql pada docker
---
## 1. Install Docker
[click here](https://www.markdownguide.org/cheat-sheet/)
## 2. Pull Image
```
docker pull php:8.1.0-fpm
docker pull mysql:8.0
docker pull nginx:alpine
```
## 3. Setup laravel
```
git clone https://github.com/madicemerlang/laravel.git laravel

cd laravel
nano Dockerfile
```
```
FROM php:8.1.0-fpm

# Copy composer.lock and composer.json
COPY composer.lock composer.json /var/www/

RUN chmod +x /usr/local/bin/install-php-extensions && \
    install-php-extensions gd xdebug

# Set working directory
WORKDIR /var/www

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    mariadb-client \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install extensions
RUN docker-php-ext-install pdo pdo_mysql

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy existing application directory contents
COPY . /var/www

# Copy existing application directory permissions
COPY --chown=www:www . /var/www

# Change current user to www
USER www

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```
buat file docker-compose.yml
```
nano docker-compose.yml
```
lalu isikan baris berikut :
```
version: '3'
services:

  #PHP Service
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: php:8.1.0-fpm
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./:/var/www
      - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - app-network

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: laravel-container
    restart: unless-stopped
    tty: true
    ports: 
      # in my local environment
      - "8080:80"
    volumes:
      - ./:/var/www
      - ./nginx/conf.d:/etc/nginx/conf.d
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: laravel
      # replace with your own mysql password here
      MYSQL_ROOT_PASSWORD: password
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - dbdata:/var/lib/mysql/
      - ./mysql/my.cnf:/etc/mysql/my.cnf
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge
#Volumes
volumes:
  dbdata:
    driver: local
```
buat folder php pada direktori laravel, lalu buat file local.ini
```
mkdir php
nano php/local.ini
```
masukkan baris berikut :
```
upload_max_filesize=40M
post_max_size=40M
```
buat folder mysql pada direktori laravel, lalu buat file my.cnf
```
mkdir mysql
nano mysql/my.cnf
```
masukkan baris berikut :
```
[mysqld]
general_log = 1
general_log_file = /var/lib/mysql/general.log
```
buat folder nginx pada direktori laravel, lalu buat folder conf.d di dalam nginx, terakhir buat file default.conf
```
mkdir nginx
mkdir nginx/conf.d
nano nginx/conf.d/default.conf
```
isikan baris berikut :
```
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```
# sekarang kita akan menghubungkan laravel dengan database
```
cp .env.example .env
nano .env
```
rubah baris berikut :
```
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=password
```
setelah itu lakukan build pada docker
```
docker-compose build
```
lalu jalankan container
```
docker-compose up -d
```
setelah suksesk menjalankan container, kita perlu menambahkan encyption key untuk laravel
```
docker-compose exec app php artisan key:generate
```
sekarang jika anda akses http://localhost:8080 akan menampilkan halaman laravel
![alt text](https://miro.medium.com/max/720/1*u0efpKLdHnbpuWs4qKWgcg.png)
sampai sini laravel sudah berjalan tetapi belum terhubung dengan database, untuk menghubungkannya sebagai berikut :
'''
docker-compose exec app php artisan config:cache
docker-compose exec app php artisan migrate
'''
jika berhasil maka akan muncul teks seperti ini :
![alt text](https://i.ibb.co/TqJpbJr/image-2022-11-03-030840103.png)
