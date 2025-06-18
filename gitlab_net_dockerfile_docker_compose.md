# Создание сложного .NET приложения с GitLab, Docker и Docker Compose

В этом руководстве я покажу, как создать более сложное .NET приложение с использованием GitLab CI/CD, Dockerfile и docker-compose, включая несколько сервисов и базу данных.

## Структура приложения

Предположим, у нас есть следующая структура:
- Web API (ASP.NET Core)
- Worker Service (фоновые задачи)
- PostgreSQL база данных
- Redis для кеширования

## 1. Dockerfile для Web API

```dockerfile
# Используем многостадийную сборку
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Копируем файлы проектов и восстанавливаем зависимости
COPY ["WebApi/WebApi.csproj", "WebApi/"]
COPY ["SharedLib/SharedLib.csproj", "SharedLib/"]
RUN dotnet restore "WebApi/WebApi.csproj"

# Копируем все файлы и собираем приложение
COPY . .
WORKDIR "/src/WebApi"
RUN dotnet build "WebApi.csproj" -c Release -o /app/build

# Публикуем приложение
FROM build AS publish
RUN dotnet publish "WebApi.csproj" -c Release -o /app/publish

# Финальный образ
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Устанавливаем инструменты для диагностики (только для разработки)
RUN apt-get update && apt-get install -y curl procps

ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:80
EXPOSE 80

ENTRYPOINT ["dotnet", "WebApi.dll"]
```

## 2. Dockerfile для Worker Service

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["WorkerService/WorkerService.csproj", "WorkerService/"]
COPY ["SharedLib/SharedLib.csproj", "SharedLib/"]
RUN dotnet restore "WorkerService/WorkerService.csproj"

COPY . .
WORKDIR "/src/WorkerService"
RUN dotnet build "WorkerService.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "WorkerService.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

ENTRYPOINT ["dotnet", "WorkerService.dll"]
```

## 3. docker-compose.yml

```yaml
version: '3.8'

services:
  webapi:
    build:
      context: .
      dockerfile: WebApi/Dockerfile
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=AppDb;User Id=postgres;Password=postgres;
      - Redis__ConnectionString=redis:6379
    depends_on:
      - db
      - redis
    networks:
      - app-network

  worker:
    build:
      context: .
      dockerfile: WorkerService/Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=AppDb;User Id=postgres;Password=postgres;
      - Redis__ConnectionString=redis:6379
    depends_on:
      - db
      - redis
    networks:
      - app-network

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: AppDb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    networks:
      - app-network

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

## 4. GitLab CI/CD (.gitlab-ci.yml)

```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_HOST: tcp://docker:2375
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  CONTAINER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
  POSTGRES_DB: testdb
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres

services:
  - docker:20.10-dind

before_script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:
  stage: build
  script:
    - docker build -f WebApi/Dockerfile -t $CONTAINER_API_IMAGE .
    - docker build -f WorkerService/Dockerfile -t $CONTAINER_WORKER_IMAGE .
    - docker push $CONTAINER_API_IMAGE
    - docker push $CONTAINER_WORKER_IMAGE

test:
  stage: test
  services:
    - postgres:15
    - redis:7
  script:
    - dotnet restore
    - dotnet build --no-restore
    - dotnet test --no-build --verbosity normal

deploy:
  stage: deploy
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
  script:
    - echo "Deploying to production..."
    - scp docker-compose.prod.yml user@server:/app
    - ssh user@server "cd /app && docker-compose -f docker-compose.prod.yml pull && docker-compose -f docker-compose.prod.yml up -d"
```

## 5. docker-compose.prod.yml для продакшена

```yaml
version: '3.8'

services:
  webapi:
    image: $CI_REGISTRY_IMAGE/webapi:latest
    restart: always
    ports:
      - "80:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=AppDb;User Id=postgres;Password=${DB_PASSWORD}
      - Redis__ConnectionString=redis:6379
    depends_on:
      - db
      - redis

  worker:
    image: $CI_REGISTRY_IMAGE/worker:latest
    restart: always
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=AppDb;User Id=postgres;Password=${DB_PASSWORD}
      - Redis__ConnectionString=redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: AppDb
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7
    restart: always

volumes:
  postgres-data:
```

## Советы по настройке

1. **Секреты**: Используйте GitLab CI/CD variables или Docker secrets для хранения паролей и других конфиденциальных данных.

2. **Масштабирование**: В продакшене вы можете масштабировать сервисы:
   ```bash
   docker-compose up -d --scale webapi=3 --scale worker=2
   ```

3. **Здоровье сервисов**: Добавьте healthcheck в ваши сервисы:
   ```yaml
   healthcheck:
     test: ["CMD", "curl", "-f", "http://localhost/health"]
     interval: 30s
     timeout: 10s
     retries: 3
   ```

4. **Логирование**: Настройте централизованное логирование с помощью ELK или аналогичного стека.

5. **Мониторинг**: Добавьте Prometheus и Grafana для мониторинга приложения.

Этот пример демонстрирует более сложную конфигурацию .NET приложения с использованием Docker и GitLab CI/CD. Вы можете адаптировать его под свои конкретные нужды.