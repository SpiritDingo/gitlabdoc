# Сложное PHP-приложение с GitLab, PHP-FPM, Dockerfile и Docker Compose

В этом руководстве я покажу, как создать более сложное PHP-приложение с использованием GitLab CI/CD, PHP-FPM, Dockerfile и Docker Compose.

## Структура проекта

```
my-php-app/
├── .gitlab-ci.yml
├── docker-compose.yml
├── docker/
│   ├── nginx/
│   │   ├── nginx.conf
│   │   └── sites/
│   │       └── app.conf
│   ├── php-fpm/
│   │   ├── Dockerfile
│   │   ├── php.ini
│   │   └── www.conf
│   └── mysql/
│       └── my.cnf
├── src/
│   ├── index.php
│   ├── config/
│   └── ...
└── .env
```

## Dockerfile для PHP-FPM

`docker/php-fpm/Dockerfile`:
```dockerfile
# Используем официальный образ PHP-FPM с нужной версией
FROM php:8.2-fpm

# Установка зависимостей
RUN apt-get update && apt-get install -y \
    git \
    libzip-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libssl-dev \
    libonig-dev \
    unzip \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) gd pdo_mysql zip opcache mbstring

# Установка Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Копируем конфигурационные файлы
COPY php.ini /usr/local/etc/php/conf.d/custom.ini
COPY www.conf /usr/local/etc/php-fpm.d/www.conf

# Устанавливаем рабочую директорию
WORKDIR /var/www/html

# Копируем зависимости composer первыми для кэширования
COPY src/composer.json src/composer.lock ./
RUN composer install --no-autoloader --no-scripts --no-dev

# Копируем остальные файлы приложения
COPY src/ .

# Генерируем автозагрузку и запускаем скрипты
RUN composer dump-autoload --optimize && composer run-script post-install-cmd

# Устанавливаем права
RUN chown -R www-data:www-data /var/www/html/storage
RUN chmod -R 755 /var/www/html/storage

# Открываем порт 9000 для PHP-FPM
EXPOSE 9000

CMD ["php-fpm"]
```

## Конфигурация Nginx

`docker/nginx/sites/app.conf`:
```nginx
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

## Docker Compose

`docker-compose.yml`:
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: docker/php-fpm/Dockerfile
    container_name: app
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - ./src:/var/www/html
      - ./docker/php-fpm/php.ini:/usr/local/etc/php/conf.d/custom.ini
    networks:
      - app-network
    depends_on:
      - db

  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/nginx/sites:/etc/nginx/conf.d
    networks:
      - app-network
    depends_on:
      - app

  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file:
      - .env
    environment:
      - MYSQL_DATABASE=${DB_DATABASE}
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${DB_PASSWORD}
      - MYSQL_USER=${DB_USERNAME}
    volumes:
      - dbdata:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    ports:
      - "3306:3306"
    networks:
      - app-network

  redis:
    image: redis:alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    networks:
      - app-network
    volumes:
      - redisdata:/data

volumes:
  dbdata:
    driver: local
  redisdata:
    driver: local

networks:
  app-network:
    driver: bridge
```

## GitLab CI/CD

`.gitlab-ci.yml`:
```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

services:
  - docker:dind

before_script:
  - docker info

build:
  stage: build
  script:
    - docker-compose build
  only:
    - master
    - develop

test:
  stage: test
  script:
    - docker-compose up -d
    - docker-compose exec -T app php artisan migrate --force
    - docker-compose exec -T app php artisan test
  only:
    - merge_requests
    - master
    - develop

deploy_production:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - docker-compose -f docker-compose.prod.yml up -d --build
  environment:
    name: production
    url: https://your-production-url.com
  only:
    - master

deploy_staging:
  stage: deploy
  script:
    - echo "Deploying to staging..."
    - docker-compose -f docker-compose.staging.yml up -d --build
  environment:
    name: staging
    url: https://your-staging-url.com
  only:
    - develop
```

## Файл окружения

`.env`:
```ini
# Application
APP_NAME=MyPHPApp
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8080

# Database
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=myphpapp
DB_USERNAME=root
DB_PASSWORD=secret

# Redis
REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379

# Mail
MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=hello@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

## Дополнительные настройки

1. **PHP-FPM конфигурация** (`docker/php-fpm/www.conf`):
```ini
[www]
user = www-data
group = www-data
listen = 9000
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
```

2. **PHP.ini настройки** (`docker/php-fpm/php.ini`):
```ini
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
date.timezone = UTC
opcache.enable=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.fast_shutdown=1
```

## Развертывание

1. Клонируйте репозиторий GitLab
2. Создайте `.env` файл и настройте окружение
3. Запустите приложение:
```bash
docker-compose up -d --build
```

4. Приложение будет доступно по адресу `http://localhost:8080`

## Особенности этой конфигурации

1. **Многосервисная архитектура**: PHP-FPM, Nginx, MySQL, Redis
2. **Оптимизированный Dockerfile** с кэшированием зависимостей Composer
3. **Полный CI/CD пайплайн** в GitLab с этапами сборки, тестирования и деплоя
4. **Разделение окружений** (production, staging)
5. **Оптимизированные настройки PHP** с включенным OPCache
6. **Правильная настройка прав** для PHP-FPM
7. **Поддержка Redis** для кэширования и очередей

Эта конфигурация подходит для средних и крупных PHP-приложений с использованием современных практик DevOps.