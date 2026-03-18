Ниже приведён пример пайплайна GitLab CI/CD для доставки PHP-кода на удалённый сервер. Пайплайн включает типовые этапы: установка зависимостей, тестирование (опционально) и деплой.

Предварительные требования

1. Сервер с доступом по SSH и установленным rsync (для эффективной синхронизации файлов).
2. Переменные CI/CD в проекте GitLab (Settings → CI/CD → Variables):
   · SSH_PRIVATE_KEY — приватный SSH-ключ для подключения к серверу.
   · SSH_KNOWN_HOSTS — отпечаток хоста (можно получить через ssh-keyscan your-server.com).
   · DEPLOY_USER — пользователь на сервере.
   · DEPLOY_HOST — адрес сервера.
   · DEPLOY_PATH — абсолютный путь к папке на сервере, куда будет разворачиваться код (например, /var/www/html).

Все переменные должны быть защищены (protected) и, при необходимости, замаскированы (masked).

Пример .gitlab-ci.yml

```yaml
stages:
  - build
  - test
  - deploy

variables:
  COMPOSER_ALLOW_SUPERUSER: "1"

# Кеширование vendor для ускорения следующих запусков
cache:
  paths:
    - vendor/

before_script:
  # Устанавливаем Composer зависимости, если нет vendor
  - if [ ! -d "vendor" ]; then composer install --no-progress --prefer-dist; fi

# Этап сборки (можно пропустить, если не требуется компиляция)
build:
  stage: build
  script:
    - echo "Сборка не требуется для PHP, зависимости уже установлены"
  artifacts:
    paths:
      - vendor/
    expire_in: 1 hour

# Этап тестирования (запуск линтера, PHPUnit и т.д.)
test:
  stage: test
  script:
    - vendor/bin/phpcs --standard=PSR12 src/   # пример кодового стиля
    - vendor/bin/phpunit tests/                 # если есть тесты
  only:
    - merge_requests
    - main

# Этап деплоя
deploy:
  stage: deploy
  before_script:
    # Устанавливаем ssh-agent и добавляем ключ
    - apt-get update -qq && apt-get install -qq -y openssh-client rsync
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - rsync -avz --delete --exclude '.git' --exclude '.gitlab' --exclude 'README.md' ./ $DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_PATH/
  environment:
    name: production   # или staging, в зависимости от ветки
    url: http://your-domain.com
  only:
    - main            # деплоим только при пушах в main
  when: manual        # опционально: требовать ручного подтверждения
```

Пояснения

· Этапы: build (сборка зависимостей), test (проверка кода), deploy (доставка).
· Кеширование: папка vendor сохраняется между запусками, что ускоряет выполнение.
· before_script в общем блоке запускается для всех джоб, если не переопределён. Здесь мы устанавливаем зависимости (если их ещё нет).
· Джоба deploy:
  · Устанавливает openssh-client и rsync (если раннер использует образ, где их нет; если используется официальный образ php:cli или подобный, это необходимо).
  · Настраивает SSH-агент с приватным ключом и known_hosts.
  · Синхронизирует текущую директорию с удалённой через rsync:
    · -a — архивный режим (сохраняет права, symlink и т.д.)
    · -v — подробный вывод
    · -z — сжатие
    · --delete — удаляет файлы на сервере, которых нет в репозитории (осторожно!)
    · --exclude — исключает ненужные для продакшена файлы (например, .git, README).
  · Привязан к ветке main и требует ручного запуска (when: manual) — это защита от случайного деплоя.

Альтернативный подход (Docker)

Если ваш сервер использует Docker, можно собирать образ и отправлять его в registry, а затем на сервере выполнять docker pull и перезапускать контейнер. Пример:

```yaml
deploy:
  stage: deploy
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    # Далее по SSH подключаемся к серверу и запускаем новый контейнер
  only:
    - main
```

Важные замечания

· Никогда не храните SSH-ключи и пароли в репозитории. Используйте переменные CI/CD.
· Для production рекомендуется добавлять дополнительные проверки (например, перед деплоем запускать smoke-тесты).
· Если нужно деплоить на несколько окружений (staging, production), можно использовать разные ветки и правила (only/except).

Этот пайплайн — база, которую можно адаптировать под конкретный проект: добавить миграции БД, очистку кеша, перезагрузку PHP-FPM и т.д.