```yaml
# .gitlab-ci.yml

# Без указания образа — используется окружение раннера (shell executor),
# где уже установлен Ansible и доступен в PATH.
# Раннер должен иметь доступ к GitLab (для установки ролей из других проектов).

variables:
  ANSIBLE_HOST_KEY_CHECKING: "false"
  INVENTORY_FILE: "hosts.yml"          # файл inventory в репозитории
  PLAYBOOK_FILE: "playbook.yml"        # плейбук, принимающий список ролей
  ROLES: ""                            # список ролей (переопределяется в job)
  ANSIBLE_ROLES_PATH: "roles"          # каталог, куда устанавливаются роли

# Скрытый шаблон: установка зависимостей и запуск ansible-playbook
.ansible_roles_job: &ansible_roles_job
  stage: deploy
  before_script:
    - echo "Установка ролей из requirements.yml..."
    # Устанавливаем роли из других GitLab-проектов (или внешних git-репозиториев)
    - ansible-galaxy install -r requirements.yml -p "$ANSIBLE_ROLES_PATH" --force
  script:
    - echo "Запуск ролей: $ROLES"
    # Передаём список ролей в плейбук через extra-vars
    - ansible-playbook -i "$INVENTORY_FILE" "$PLAYBOOK_FILE" \
        --extra-vars "roles_to_run='$ROLES'"
  when: manual  # каждый job запускается вручную
  tags:
    - ansible-runner  # замените на свой тег раннера

# ======================================================
# Примеры job'ов с разными наборами ролей
# ======================================================

# Шаг 1: только базовая настройка
deploy_common:
  <<: *ansible_roles_job
  variables:
    ROLES: "common"

# Шаг 2: развёртывание приложения (несколько ролей в одном шаге)
deploy_app:
  <<: *ansible_roles_job
  variables:
    ROLES: "common, app, monitoring"

# Шаг 3: база данных и бэкапы
deploy_database:
  <<: *ansible_roles_job
  variables:
    ROLES: "common, postgresql, backup"

# Шаг 4: веб-сервер и файрвол
deploy_web:
  <<: *ansible_roles_job
  variables:
    ROLES: "common, nginx, firewall"
```

---

Что нужно подготовить в репозитории

1. Файл requirements.yml

Содержит список ролей, которые будут устанавливаться из других GitLab-проектов.
Для приватных проектов используйте переменную CI_JOB_TOKEN (автоматически доступна в CI) для аутентификации при клонировании.

```yaml
# requirements.yml
- src: git+https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/group/ansible-role-common.git
  name: common
  version: main

- src: git+https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/group/ansible-role-app.git
  name: app
  version: main

- src: git+https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/group/ansible-role-monitoring.git
  name: monitoring
  version: main

# ... и так далее для всех ролей
```

Примечание: если роли находятся в публичных репозиториях, можно использовать обычный URL без токена.
Также можно указать конкретную ветку, тег или коммит через version.

2. Плейбук playbook.yml

Должен принимать переменную roles_to_run (список ролей через запятую) и выполнять их.

```yaml
# playbook.yml
- hosts: all
  gather_facts: yes
  tasks:
    - name: Include specified roles
      include_role:
        name: "{{ item }}"
      loop: "{{ roles_to_run.split(',') | map('trim') | list }}"
```

3. Inventory hosts.yml

Файл с перечнем целевых хостов (можно использовать переменные окружения или динамический inventory при необходимости).

```yaml
# hosts.yml
all:
  hosts:
    server1:
      ansible_host: 192.168.1.10
      ansible_user: deploy
    # ...
```

---

Как это работает

1. При ручном запуске любого job'а GitLab выполняет before_script:
   · ansible-galaxy install -r requirements.yml -p roles/ --force скачивает все роли из указанных репозиториев в локальный каталог roles/.
2. Затем выполняется script:
   · Запускается ansible-playbook с нужным inventory и плейбуком.
   · Через --extra-vars передаётся переменная roles_to_run, содержащая список ролей (например, "common, app, monitoring").
   · Плейбук последовательно включает каждую роль из списка.

Таким образом, один job может выполнить сразу несколько ролей, а каждый job запускается вручную из интерфейса GitLab.