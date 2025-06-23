# Пример использования GitLab CI с PHP-FPM, Docker, Apache 2

Вот пример настройки GitLab CI/CD для PHP-приложения с использованием PHP-FPM, Docker и Apache 2.

## Структура проекта

```
my-php-app/
├── .gitlab-ci.yml
├── docker-compose.yml
├── Dockerfile
├── apache/
│   └── 000-default.conf
└── src/
    └── index.php
```

## 1. Dockerfile для PHP-FPM

```dockerfile
# Dockerfile
FROM php:8.2-fpm

# Установка зависимостей
RUN apt-get update && apt-get install -y \
    libzip-dev \
    zip \
    && docker-php-ext-install pdo_mysql zip

# Установка Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Копирование файлов приложения
WORKDIR /var/www/html
COPY src/ .

# Установка зависимостей Composer (если есть)
# COPY composer.json composer.lock ./
# RUN composer install --no-dev --no-scripts --no-autoloader && \
#     composer clear-cache

# Настройка прав
RUN chown -R www-data:www-data /var/www/html
```

## 2. Конфигурация Apache (000-default.conf)

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # Настройки для PHP-FPM
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php-fpm:9000/var/www/html/$1
</VirtualHost>
```

## 3. docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    image: httpd:2.4
    ports:
      - "8080:80"
    volumes:
      - ./apache/000-default.conf:/usr/local/apache2/conf/httpd.conf
      - ./src:/var/www/html
    depends_on:
      - php-fpm
    networks:
      - app-network

  php-fpm:
    build: .
    volumes:
      - ./src:/var/www/html
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## 4. .gitlab-ci.yml

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - master

test:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker-compose up -d
    - apk add --no-cache curl
    - sleep 10 # Даем время контейнерам запуститься
    - curl -I http://web:8080 | grep "200 OK"
    - docker-compose down

deploy:
  stage: deploy
  image: alpine:latest
  script:
    - apk add --no-cache openssh-client rsync
    - echo "$SSH_PRIVATE_KEY" > ssh_key
    - chmod 600 ssh_key
    - rsync -avz -e "ssh -i ssh_key -o StrictHostKeyChecking=no" ./src/ $DEPLOY_USER@$DEPLOY_SERVER:/var/www/html/
  only:
    - master
```

## Дополнительные настройки

1. **Переменные GitLab CI/CD**:
   - `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`, `CI_REGISTRY_IMAGE` - для Docker Registry
   - `SSH_PRIVATE_KEY` - приватный ключ для деплоя
   - `DEPLOY_USER`, `DEPLOY_SERVER` - данные сервера для деплоя

2. **Альтернативный вариант для продакшена**:
   Для продакшена лучше использовать отдельные Dockerfile для Apache и PHP-FPM, а также добавить SSL.

3. **Оптимизация**:
   - Кэширование слоев Docker
   - Параллельное выполнение тестов
   - Использование артефактов между этапами

Этот пример предоставляет базовую настройку, которую можно расширить в зависимости от потребностей вашего проекта.