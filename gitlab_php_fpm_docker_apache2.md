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


# GitLab CI/CD для PHP-FPM с Docker, Docker Compose и Apache 2

Вот пример настройки CI/CD для PHP-приложения с использованием PHP-FPM, Docker, Docker Compose и Apache 2.

## 1. Dockerfile для PHP-FPM

Создайте `Dockerfile` для вашего PHP-приложения:

```dockerfile
# Используем официальный образ PHP-FPM
FROM php:8.2-fpm

# Установка зависимостей
RUN apt-get update && apt-get install -y \
    libzip-dev \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip

# Установка расширений PHP
RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip

# Установка Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Рабочая директория
WORKDIR /var/www/html

# Копируем файлы приложения
COPY . .

# Установка зависимостей Composer (без dev-зависимостей)
RUN composer install --no-dev --optimize-autoloader

# Права на запись
RUN chown -R www-data:www-data /var/www/html/storage
RUN chmod -R 775 /var/www/html/storage
```

## 2. Docker-compose.yml

Создайте `docker-compose.yml` для локальной разработки и тестирования:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www/html
    environment:
      - DB_HOST=db
      - DB_DATABASE=app_db
      - DB_USERNAME=root
      - DB_PASSWORD=secret
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=app_db
    volumes:
      - db_data:/var/lib/mysql

  webserver:
    image: httpd:2.4
    ports:
      - "8080:80"
    volumes:
      - ./:/var/www/html
      - ./docker/apache/vhost.conf:/usr/local/apache2/conf/extra/vhost.conf
    depends_on:
      - app
    links:
      - app

volumes:
  db_data:
```

## 3. Конфигурация Apache (vhost.conf)

Создайте файл `docker/apache/vhost.conf`:

```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html/public

    <Directory /var/www/html/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://app:9000/var/www/html/public/$1
</VirtualHost>
```

## 4. GitLab CI/CD (.gitlab-ci.yml)

Вот пример конфигурации `.gitlab-ci.yml`:

```yaml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG-$CI_COMMIT_SHORT_SHA

# Используем Docker-in-Docker (dind) сервис
services:
  - docker:dind

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

test:
  stage: test
  image: php:8.2
  script:
    - apt-get update && apt-get install -y zip unzip git
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install
    - vendor/bin/phpunit

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - master
    - develop

deploy:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y )'
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan $SERVER_IP >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - ssh $SSH_USER@$SERVER_IP "docker pull $IMAGE_TAG && docker-compose -f /path/to/your/docker-compose.prod.yml up -d"
  only:
    - master
```

## 5. Docker-compose.prod.yml для продакшена

Создайте отдельный файл `docker-compose.prod.yml` для продакшена:

```yaml
version: '3.8'

services:
  app:
    image: $IMAGE_TAG
    restart: always
    environment:
      - DB_HOST=db
      - DB_DATABASE=app_db
      - DB_USERNAME=app_user
      - DB_PASSWORD=prod_password
    depends_on:
      - db

  db:
    image: mysql:8.0
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=prod_root_password
      - MYSQL_DATABASE=app_db
      - MYSQL_USER=app_user
      - MYSQL_PASSWORD=prod_password
    volumes:
      - db_data:/var/lib/mysql

  webserver:
    image: httpd:2.4
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./docker/apache/vhost.conf:/usr/local/apache2/conf/extra/vhost.conf
    depends_on:
      - app

volumes:
  db_data:
```

## Настройка переменных в GitLab

Не забудьте добавить следующие переменные в настройках GitLab CI/CD:
- `CI_REGISTRY_USER` - имя пользователя Docker Registry
- `CI_REGISTRY_PASSWORD` - пароль Docker Registry
- `SSH_PRIVATE_KEY` - приватный SSH-ключ для доступа к серверу
- `SSH_USER` - пользователь SSH
- `SERVER_IP` - IP сервера для деплоя

Эта конфигурация обеспечит:
1. Тестирование PHP-приложения
2. Сборку Docker-образа
3. Деплой на продакшен-сервер с использованием Docker Compose
4. Автоматическое обновление приложения при пуше в master-ветку