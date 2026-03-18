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


-----------------------------


Мы уже рассмотрели базовый пайплайн для доставки PHP‑кода на сервер. Теперь давайте его улучшим – добавим безопасность, производительность, дополнительные проверки и удобство эксплуатации.

1. Безопасность

1.1. SSH‑ключи

Вместо хранения приватного ключа в переменной можно использовать встроенный механизм GitLab:
Settings → Repository → Deploy keys или CI/CD → Variables (как мы делали), но с дополнительной защитой:

· Ограничьте ключ на сервере командой в ~/.ssh/authorized_keys:
  ```
  command="rsync --server ...",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc,no-X11-forwarding ssh-rsa AAAA...
  ```
  Так ключ можно использовать только для rsync.
· Храните ключ в защищённой переменной, доступной только для protected веток (main).

1.2. Права на сервере

Создайте отдельного системного пользователя для деплоя (например, deploy), который имеет права только на целевые директории. Не используйте root.

1.3. Проверка кода перед деплоем

Запускайте статический анализатор и тесты до этапа деплоя. Если они упадут – деплой не выполнится.

2. Производительность

2.1. Кеширование зависимостей Composer

Мы уже добавили cache: paths: vendor/. Улучшим, указав ключи для разных веток:

```yaml
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - vendor/
```

2.2. Использование готовых Docker‑образов

В базовом пайплайне мы устанавливали openssh-client и rsync в каждом запуске. Лучше выбрать образ, где они уже есть, например:

· alpine:latest (потом доустановить rsync, openssh – всё равно легче)
· Собственный образ на основе php:cli с предустановленными инструментами.

Пример использования php:cli (в нём нет rsync, но есть ssh):

```yaml
deploy:
  image: php:cli
  before_script:
    - apt-get update && apt-get install -y rsync openssh-client
  ...
```

2.3. Параллельные джобы

Если у вас много тестов или статических анализаторов, распараллельте их:

```yaml
test:phpcs:
  stage: test
  script: vendor/bin/phpcs

test:phpstan:
  stage: test
  script: vendor/bin/phpstan analyse

test:phpunit:
  stage: test
  script: vendor/bin/phpunit
```

3. Дополнительные этапы

3.1. Статический анализ и линтинг

Добавим в test этап:

· phpcs – проверка стиля кода
· phpstan / psalm – анализ типов
· phpmd – поиск «запахов» кода

3.2. Миграции базы данных

Если проект использует БД, миграции стоит запускать после выкладки файлов, но до переключения симлинка (если используется стратегия релизов).
Пример с использованием rsync и последовательным переключением:

```yaml
deploy:
  script:
    - rsync -avz --delete ./ $DEPLOY_USER@$DEPLOY_HOST:/tmp/new_release/
    - ssh $DEPLOY_USER@$DEPLOY_HOST "cd /tmp/new_release && php artisan migrate --force"
    - ssh $DEPLOY_USER@$DEPLOY_HOST "ln -sfn /tmp/new_release $DEPLOY_PATH/current"
```

3.3. Очистка кеша приложения

После деплоя часто нужно сбросить кеш (Laravel: php artisan cache:clear, Symfony: php bin/console cache:clear). Это можно сделать через SSH:

```yaml
after_script:
  - ssh $DEPLOY_USER@$DEPLOY_HOST "cd $DEPLOY_PATH && php artisan optimize:clear"
```

3.4. Smoke‑тесты после деплоя

Убедитесь, что сайт отвечает 200 OK:

```yaml
after_script:
  - curl --fail https://your-site.com || exit 1
```

4. Управление окружениями

4.1. Разные окружения

Настройте деплой на staging для всех веток, кроме main, а на production – только для тегов или main с ручным подтверждением.

```yaml
deploy:staging:
  stage: deploy
  environment: staging
  only:
    - branches
  except:
    - main
  when: manual   # можно автоматически

deploy:production:
  stage: deploy
  environment: production
  only:
    - main
    - tags
  when: manual
```

4.2. Review Apps (если используется Kubernetes или динамические серверы)

Для каждой ветки можно создавать временное окружение. Это требует более сложной настройки, но сильно упрощает тестирование MR.

5. Откаты и бекапы

Перед деплоем создавайте резервную копию текущей версии:

```yaml
before_script:
  - ssh $DEPLOY_USER@$DEPLOY_HOST "cp -r $DEPLOY_PATH $DEPLOY_PATH.backup-$(date +%Y%m%d-%H%M%S)"
```

А для быстрого отката можно хранить несколько последних релизов и переключать симлинк.

6. Использование Deployer

Deployer – популярный инструмент для деплоя PHP‑приложений. Он умеет управлять релизами, миграциями, кешем и откатами. Пример интеграции в GitLab CI:

```yaml
deploy:
  stage: deploy
  image: deployer/deployer:latest
  script:
    - dep deploy production
  only:
    - main
```

Переменные окружения (хост, пользователь, ключ) передаются через переменные GitLab.

7. Уведомления

7.1. Slack/Telegram

После успешного или неудачного деплоя отправляйте уведомление. Пример для Slack через вебхук:

```yaml
after_script:
  - 'curl -X POST -H "Content-type: application/json" --data "{\"text\":\"Деплой на production завершён\"}" $SLACK_WEBHOOK_URL'
```

8. Пример улучшенного .gitlab-ci.yml (фрагмент)

```yaml
stages:
  - test
  - deploy

variables:
  COMPOSER_ALLOW_SUPERUSER: "1"
  DEPLOY_USER: "deploy"
  DEPLOY_HOST: "example.com"
  DEPLOY_PATH: "/var/www/project"

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - vendor/

before_script:
  - composer install --no-progress --prefer-dist

test:phpcs:
  stage: test
  script: vendor/bin/phpcs --standard=PSR12 src/

test:phpstan:
  stage: test
  script: vendor/bin/phpstan analyse src --level=5

test:phpunit:
  stage: test
  script: vendor/bin/phpunit

deploy:production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client rsync
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
    - mkdir -p ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
  script:
    - rsync -avz --delete --exclude='.git' --exclude='.gitlab' --exclude='tests/' ./ $DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_PATH/
    - ssh $DEPLOY_USER@$DEPLOY_HOST "cd $DEPLOY_PATH && php artisan migrate --force"
  after_script:
    - curl --fail https://example.com || exit 1
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual
```

Заключение

Эти улучшения сделают ваш пайплайн:

· безопаснее – ограниченные ключи, проверки до деплоя;
· быстрее – кеши, готовые образы, параллельные задачи;
· надёжнее – миграции, бекапы, smoke‑тесты;
· удобнее – уведомления, ручной триггер для продакшена.

Адаптируйте предложенные идеи под свой проект, и процесс доставки PHP‑кода станет по‑настоящему профессиональным.


Ключевое слово gitlab-runner — это программа, которая непосредственно выполняет задачи (jobs), описанные в файле .gitlab-ci.yml. Раннер можно установить на том же сервере, где лежит код, или на отдельной машине. Рассмотрим настройку на примере Ubuntu/Debian, так как это самый частый вариант для PHP-проектов.

---

1. Установка GitLab Runner

Подключите официальный репозиторий GitLab Runner и установите пакет:

```bash
# Добавляем репозиторий GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash

# Устанавливаем сам раннер
sudo apt-get install gitlab-runner

# Проверяем, что сервис запущен
sudo systemctl status gitlab-runner
```

Примечание: если вы используете CentOS/RHEL, замените .deb.sh на .rpm.sh и apt на yum. Для Windows нужно скачать бинарник и установить его как службу .

2. Регистрация раннера в проекте

Раннер нужно привязать к вашему GitLab-проекту. Для этого понадобятся URL и токен регистрации.

Где взять токен?

1. Зайдите в свой проект на GitLab.
2. Settings → CI/CD → Runners.
3. В разделе Specific runners вы увидите URL (например, https://gitlab.com/) и registration token.

Теперь выполните регистрацию:

```bash
sudo gitlab-runner register
```

В процессе регистрации вас спросят:

· URL – вставьте URL из настроек.
· Token – вставьте токен.
· Описание – например, php-production-runner.
· Теги – можно указать, например, production, php (потом их можно использовать в .gitlab-ci.yml для выбора конкретного раннера).
· Executor – выберите shell, так как мы будем выполнять команды прямо на сервере (это проще всего для деплоя PHP) .

После регистрации раннер появится в списке Settings → CI/CD → Runners с зелёным индикатором.

3. Настройка прав для пользователя gitlab-runner

Все задачи будут выполняться от системного пользователя gitlab-runner. Ему нужно дать права на те директории, куда будет разворачиваться код.

Создайте целевую папку (если её нет) и назначьте владельца:

```bash
sudo mkdir -p /var/www/yourapp
sudo chown -R gitlab-runner:gitlab-runner /var/www/yourapp
```

Если вы планируете использовать rsync или scp для деплоя на другой сервер, пользователю gitlab-runner также потребуется SSH-ключ для беспарольного доступа к тому серверу :

```bash
# Переключитесь на пользователя gitlab-runner
sudo su - gitlab-runner

# Сгенерируйте ssh-ключ (если его нет)
ssh-keygen -t rsa -b 2048

# Скопируйте публичный ключ на целевой сервер
ssh-copy-id deploy_user@target_server_ip
```

4. Подготовка файла .gitlab-ci.yml

Теперь в корне вашего репозитория нужно создать файл .gitlab-ci.yml с описанием пайплайна. Ниже — пример для PHP-проекта с двумя этапами: тестирование и деплой. Обратите внимание на секцию tags — она должна соответствовать тегу, указанному при регистрации раннера.

```yaml
stages:
  - test
  - deploy

# Переменные, доступные на всех этапах
variables:
  COMPOSER_ALLOW_SUPERUSER: "1"

# Кеширование vendor для ускорения
cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - vendor/

before_script:
  - composer install --no-progress --prefer-dist

# Этап тестирования (пример с PHPUnit)
test:
  stage: test
  tags:
    - php          # используем раннер с тегом php
  script:
    - vendor/bin/phpunit --testdox
  only:
    - merge_requests
    - main

# Этап деплоя на продакшн
deploy:
  stage: deploy
  tags:
    - production    # используем тот же раннер, но с другим тегом (можно и один тег)
  script:
    # Синхронизируем файлы с целевым сервером через rsync
    - rsync -avz --delete --exclude='.git' --exclude='.gitlab' --exclude='tests/' ./ /var/www/yourapp/
    # Пример дополнительных команд: миграции, очистка кеша
    - cd /var/www/yourapp && php artisan migrate --force
  environment:
    name: production
    url: https://yourapp.com
  only:
    - main
  when: manual       # деплой только по ручному подтверждению
```

Объяснение ключевых моментов:

· tags – указываем, какой раннер должен выполнять задачу. Если не указать, задачу может взять любой свободный раннер (в том числе общие раннеры GitLab.com) .
· script – команды, которые выполняются последовательно. В случае деплоя мы копируем файлы через rsync. Убедитесь, что rsync установлен на сервере с раннером (sudo apt install rsync).
· only: main – задачи запускаются только для ветки main.
· when: manual – этап требует ручного запуска через интерфейс GitLab (кнопка "play").

5. Переменные окружения для безопасности

Никогда не храните пароли и ключи в самом файле .gitlab-ci.yml. Используйте переменные CI/CD:

1. Перейдите в Settings → CI/CD → Variables.
2. Добавьте переменные, например:
   · SSH_PRIVATE_KEY – приватный ключ для доступа к серверу деплоя (если используется).
   · DB_PASSWORD – пароль базы данных.

Внутри скрипта обращайтесь к ним как к обычным переменным окружения: $SSH_PRIVATE_KEY.

6. Запуск и проверка

1. Закоммитьте .gitlab-ci.yml в корневую папку репозитория и запушите в ветку main.
2. Перейдите в CI/CD → Pipelines — вы увидите запущенный пайплайн.
3. Если всё настроено верно, задачи начнут выполняться на вашем раннере. В логах будет видно, какой именно раннер взял задачу (например, Running on your-server-name).
4. Для production-деплоя потребуется нажать кнопку play напротив задачи deploy (так как мы указали when: manual).

Дополнительные рекомендации

· Если вы используете Docker – можно выбрать executor docker при регистрации раннера. Тогда каждая задача будет выполняться в чистом контейнере. Это удобно для изоляции, но сложнее в настройке (потребуется проброс SSH-ключей и томов) .
· Мониторинг – следите за логами раннера: sudo journalctl -u gitlab-runner -f.
· Обновление – периодически обновляйте раннер: sudo apt-get update && sudo apt-get upgrade gitlab-runner.

Таким образом, установив и зарегистрировав GitLab Runner с executor'ом shell, вы получаете полный контроль над процессом деплоя PHP-кода: от тестирования до выкладки на сервер.