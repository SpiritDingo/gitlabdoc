Вот пример .gitlab-ci.yml для сборки и деплоя Airflow с Docker Compose:

```yaml
stages:
  - build
  - deploy

variables:
  # Имя образа в GitLab Registry
  IMAGE_NAME: $CI_REGISTRY_IMAGE/airflow
  # Используем SHA коммита как тег для точности
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  # Или можно использовать ветку
  # IMAGE_TAG: $CI_COMMIT_REF_SLUG
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2

# Сервис Docker-in-Docker для сборки
services:
  - docker:20.10-dind

# Кэширование для ускорения сборок
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .cache/

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# 1. Сборка образа
build:
  stage: build
  image: docker:20.10
  script:
    # Сборка образа Airflow
    - docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .
    # Пуш в GitLab Registry
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
  artifacts:
    paths:
      - docker-compose.prod.yml
    expire_in: 1 week
  only:
    - master
    - develop
    - tags

# 2. Деплой на другой сервер
deploy:
  stage: deploy
  image: alpine:latest
  before_script:
    # Устанавливаем SSH клиент и Docker клиент
    - apk add --no-cache openssh-client docker-cli
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo "Host *" > ~/.ssh/config
    - echo "  StrictHostKeyChecking no" >> ~/.ssh/config
    # Логинимся в GitLab Registry с целевого сервера
    - ssh $DEPLOY_USER@$DEPLOY_SERVER "docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY"
  script:
    # Копируем docker-compose файл на сервер
    - scp docker-compose.prod.yml $DEPLOY_USER@$DEPLOY_SERVER:/opt/airflow/
    
    # Выполняем деплой на удаленном сервере
    - |
      ssh $DEPLOY_USER@$DEPLOY_SERVER "
        cd /opt/airflow &&
        
        # Останавливаем текущий контейнер
        docker-compose -f docker-compose.prod.yml down || true
        
        # Обновляем образ
        docker pull $IMAGE_NAME:$IMAGE_TAG
        
        # Запускаем с новым образом
        docker-compose -f docker-compose.prod.yml up -d
        
        # Очистка старых образов
        docker image prune -f
      "
  environment:
    name: production
    url: https://airflow.yourdomain.com
  only:
    - master
    - tags
```

Необходимые файлы в репозитории:

1. Dockerfile для Airflow:

```dockerfile
FROM apache/airflow:2.7.0-python3.9

USER root

# Установка дополнительных системных пакетов
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER airflow

# Копирование requirements.txt если есть
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копирование DAG'ов и конфигурации
COPY dags/ /opt/airflow/dags/
COPY config/ /opt/airflow/config/
COPY scripts/ /opt/airflow/scripts/
```

2. docker-compose.yml для разработки:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  airflow-webserver:
    build: .
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./config:/opt/airflow/config
    ports:
      - "8080:8080"
    command: webserver
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  airflow-scheduler:
    build: .
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./config:/opt/airflow/config
    command: scheduler

  airflow-worker:
    build: .
    restart: always
    depends_on:
      - airflow-scheduler
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./config:/opt/airflow/config
    command: celery worker

  airflow-init:
    build: .
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    command: >
      bash -c "
      airflow db init &&
      airflow users create \
        --username admin \
        --firstname Admin \
        --lastname User \
        --role Admin \
        --email admin@example.com \
        --password admin
      "

volumes:
  postgres-db-volume:
```

3. docker-compose.prod.yml для продакшена:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-airflow}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-airflow}
      POSTGRES_DB: ${POSTGRES_DB:-airflow}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - airflow-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    networks:
      - airflow-network
    restart: unless-stopped

  airflow-webserver:
    image: ${AIRFLOW_IMAGE:-registry.gitlab.com/your-project/airflow:latest}
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://${POSTGRES_USER:-airflow}:${POSTGRES_PASSWORD:-airflow}@postgres/${POSTGRES_DB:-airflow}
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://${POSTGRES_USER:-airflow}:${POSTGRES_PASSWORD:-airflow}@postgres/${POSTGRES_DB:-airflow}
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
      AIRFLOW__WEBSERVER__SECRET_KEY: ${AIRFLOW_SECRET_KEY}
    volumes:
      - dags-data:/opt/airflow/dags
      - logs-data:/opt/airflow/logs
      - plugins-data:/opt/airflow/plugins
    ports:
      - "8080:8080"
    networks:
      - airflow-network
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  airflow-scheduler:
    image: ${AIRFLOW_IMAGE:-registry.gitlab.com/your-project/airflow:latest}
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://${POSTGRES_USER:-airflow}:${POSTGRES_PASSWORD:-airflow}@postgres/${POSTGRES_DB:-airflow}
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://${POSTGRES_USER:-airflow}:${POSTGRES_PASSWORD:-airflow}@postgres/${POSTGRES_DB:-airflow}
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    volumes:
      - dags-data:/opt/airflow/dags
      - logs-data:/opt/airflow/logs
      - plugins-data:/opt/airflow/plugins
    networks:
      - airflow-network
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  airflow-worker:
    image: ${AIRFLOW_IMAGE:-registry.gitlab.com/your-project/airflow:latest}
    environment:
      AIRFLOW__CORE__EXECUTOR: CeleryExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://${POSTGRES_USER:-airflow}:${POSTGRES_PASSWORD:-airflow}@postgres/${POSTGRES_DB:-airflow}
      AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://${POSTGRES_USER:-airflow}:${POSTGRES_PASSWORD:-airflow}@postgres/${POSTGRES_DB:-airflow}
      AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
      AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    volumes:
      - dags-data:/opt/airflow/dags
      - logs-data:/opt/airflow/logs
      - plugins-data:/opt/airflow/plugins
    networks:
      - airflow-network
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

volumes:
  postgres-data:
  dags-data:
  logs-data:
  plugins-data:

networks:
  airflow-network:
    driver: bridge
```

4. .env файл для продакшена (не коммитить!):

```bash
# На продакшен сервере в /opt/airflow/.env
POSTGRES_USER=airflow_prod
POSTGRES_PASSWORD=strong_password_here
POSTGRES_DB=airflow_prod
AIRFLOW_SECRET_KEY=your_secret_key_here
AIRFLOW_IMAGE=registry.gitlab.com/your-project/airflow:latest
```

Переменные GitLab CI/CD:

Установите в GitLab (Settings → CI/CD → Variables):

1. DEPLOY_SERVER - IP/домен сервера для деплоя
2. DEPLOY_USER - пользователь для SSH подключения
3. SSH_PRIVATE_KEY - приватный ключ для доступа к серверу
4. CI_REGISTRY_USER - автоматически установлен
5. CI_REGISTRY_PASSWORD - автоматически установлен

Настройка сервера для деплоя:

1. Установите Docker и Docker Compose на целевой сервер
2. Создайте директорию для Airflow:

```bash
sudo mkdir -p /opt/airflow
sudo chown -R $USER:$USER /opt/airflow
```

1. Создайте файл .env с переменными окружения
2. Добавьте публичный ключ GitLab Runner в ~/.ssh/authorized_keys

Этот конфиг обеспечивает:

· Сборку Docker образа Airflow
· Загрузку образа в GitLab Registry
· Деплой на целевой сервер с автоматическим обновлением
· Использование Docker Compose для управления сервисами
· Поддержку PostgreSQL и Redis для Airflow