# GitLab CI/CD для деплоя PHP-приложения с PHP-FPM, Dockerfile и docker-compose

Вот полная настройка CI/CD в GitLab для автоматического развертывания PHP-приложения с использованием PHP-FPM, Dockerfile и docker-compose.

## 1. Подготовка инфраструктуры

### `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG-$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - branches

test:
  stage: test
  image: $IMAGE_TAG
  script:
    - composer install
    - vendor/bin/phpunit tests/
  needs: ["build"]
  only:
    - branches

deploy:
  stage: deploy
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  before_script:
    - apk add --no-cache openssh-client rsync
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
  script:
    - ssh $SSH_USER@$SSH_HOST "mkdir -p /opt/apps/$CI_PROJECT_NAME"
    - scp docker-compose.prod.yml $SSH_USER@$SSH_HOST:/opt/apps/$CI_PROJECT_NAME/docker-compose.yml
    - scp .env.prod $SSH_USER@$SSH_HOST:/opt/apps/$CI_PROJECT_NAME/.env
    - ssh $SSH_USER@$SSH_HOST "cd /opt/apps/$CI_PROJECT_NAME && docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY"
    - ssh $SSH_USER@$SSH_HOST "cd /opt/apps/$CI_PROJECT_NAME && docker-compose pull"
    - ssh $SSH_USER@$SSH_HOST "cd /opt/apps/$CI_PROJECT_NAME && docker-compose down && docker-compose up -d"
  environment:
    name: production
    url: https://your-app-domain.com
  only:
    - main
```

### `Dockerfile` (для PHP-FPM)

```dockerfile
FROM php:8.2-fpm

# Установка зависимостей
RUN apt-get update && apt-get install -y \
    git \
    libzip-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libssl-dev \
    libxml2-dev \
    libicu-dev \
    zlib1g-dev \
    unzip \
    && rm -rf /var/lib/apt/lists/*

# Установка расширений PHP
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install zip \
    && docker-php-ext-install intl \
    && docker-php-ext-install opcache \
    && docker-php-ext-install exif \
    && docker-php-ext-install pcntl

# Установка Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Копирование конфигураций
COPY php-fpm.conf /usr/local/etc/php-fpm.d/www.conf
COPY php.ini /usr/local/etc/php/conf.d/custom.ini

WORKDIR /var/www/html

COPY . .

RUN composer install --no-dev --optimize-autoloader

EXPOSE 9000

CMD ["php-fpm"]
```

### `docker-compose.prod.yml`

```yaml
version: '3.8'

services:
  app:
    image: ${CI_REGISTRY_IMAGE}:${IMAGE_TAG}
    restart: always
    environment:
      - APP_ENV=prod
      - APP_DEBUG=0
      - DATABASE_URL=mysql://user:pass@db:3306/dbname
    volumes:
      - ./storage:/var/www/html/storage
    depends_on:
      - db
      - redis

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./storage/logs:/var/log/nginx
    depends_on:
      - app

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:alpine
    restart: always
    volumes:
      - redis_data:/data

volumes:
  mysql_data:
  redis_data:
```

### `nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    upstream php-fpm {
        server app:9000;
    }

    server {
        listen 80;
        server_name your-app-domain.com;

        root /var/www/html/public;
        index index.php index.html;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_pass php-fpm;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
            include fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }

        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
    }
}
```

## 2. Настройка переменных окружения в GitLab

1. Перейдите в Settings → CI/CD → Variables
2. Добавьте следующие переменные:
   - `SSH_PRIVATE_KEY` - приватный SSH-ключ для доступа к серверу
   - `SSH_USER` - пользователь для SSH (например, `deploy`)
   - `SSH_HOST` - IP или домен сервера
   - `DB_ROOT_PASSWORD`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` - данные для БД
   - `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD` - для доступа к registry GitLab

## 3. Развертывание на сервере

1. Установите Docker и docker-compose на сервер
2. Скопируйте файлы `.env.prod` и `docker-compose.prod.yml` в `/opt/apps/your-app`
3. GitLab CI/CD автоматически развернет приложение при пуше в ветку `main`

## 4. Дополнительные улучшения

1. **Миграции базы данных**: Добавьте шаг в CI/CD для выполнения миграций:

```yaml
run_migrations:
  stage: deploy
  script:
    - ssh $SSH_USER@$SSH_HOST "cd /opt/apps/$CI_PROJECT_NAME && docker-compose exec app php artisan migrate --force"
  needs: ["deploy"]
```

2. **Очистка кеша**:

```yaml
clear_cache:
  stage: deploy
  script:
    - ssh $SSH_USER@$SSH_HOST "cd /opt/apps/$CI_PROJECT_NAME && docker-compose exec app php artisan cache:clear"
    - ssh $SSH_USER@$SSH_HOST "cd /opt/apps/$CI_PROJECT_NAME && docker-compose exec app php artisan view:clear"
  needs: ["deploy"]
```

3. **Health checks**:

```yaml
health_check:
  stage: deploy
  script:
    - curl -I https://your-app-domain.com
  needs: ["deploy"]
```

Эта конфигурация обеспечит полный цикл CI/CD для вашего PHP-приложения с автоматическим развертыванием через GitLab.