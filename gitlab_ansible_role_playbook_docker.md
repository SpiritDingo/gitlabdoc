Вот универсальный GitLab CI/CD шаблон для запуска Ansible через Docker с возможностью выбора ролей/плейбуков через GUI:

```yaml
stages:
  - validate
  - deploy

variables:
  # Основные переменные
  ANSIBLE_ROLES_PATH: "${CI_PROJECT_DIR}/roles"
  ANSIBLE_INVENTORY_PATH: "${CI_PROJECT_DIR}/inventory"
  DOCKER_IMAGE: "quay.io/ansible/ansible-runner:latest"
  
  # Переменные по умолчанию (можно переопределить через GUI)
  PLAYBOOK: ""
  ROLE: ""
  TAGS: ""
  SKIP_TAGS: ""
  EXTRA_VARS: ""
  LIMIT: ""
  INVENTORY: "production"
  CHECK_MODE: "false"
  VERBOSITY: "v"

# Docker executor конфигурация
image: $DOCKER_IMAGE

before_script:
  - ansible --version
  - python --version
  - |
    if [ -f "requirements.yml" ]; then
      echo "Installing Ansible dependencies..."
      ansible-galaxy install -r requirements.yml -p ${ANSIBLE_ROLES_PATH}
    fi
  - |
    if [ -f "requirements.txt" ]; then
      echo "Installing Python dependencies..."
      pip install -r requirements.txt
    fi

# Валидация Ansible файлов
validate:
  stage: validate
  script:
    - ansible-lint --version || pip install ansible-lint
    - |
      if [ -n "$PLAYBOOK" ] && [ -f "$PLAYBOOK" ]; then
        echo "Validating playbook: $PLAYBOOK"
        ansible-playbook $PLAYBOOK --syntax-check
        ansible-lint $PLAYBOOK
      fi
    - |
      if [ -n "$ROLE" ]; then
        echo "Validating role: $ROLE"
        ansible-lint ${ANSIBLE_ROLES_PATH}/$ROLE
      fi
    - yamllint .
  rules:
    - when: always

# Запуск Ansible
run_ansible:
  stage: deploy
  script:
    - |
      set -e
      
      # Базовые параметры
      ANSIBLE_CMD="ansible-playbook"
      CMD_OPTS=""
      
      # Определяем inventory
      if [ -n "$INVENTORY" ]; then
        INVENTORY_FILE="${ANSIBLE_INVENTORY_PATH}/${INVENTORY}"
        if [ -f "$INVENTORY_FILE" ]; then
          CMD_OPTS="$CMD_OPTS -i $INVENTORY_FILE"
        elif [ -f "${INVENTORY_FILE}.yml" ]; then
          CMD_OPTS="$CMD_OPTS -i ${INVENTORY_FILE}.yml"
        else
          echo "Inventory file not found: $INVENTORY_FILE"
          exit 1
        fi
      fi
      
      # Выбор цели запуска: роль или плейбук
      if [ -n "$ROLE" ]; then
        echo "Running Ansible role: $ROLE"
        TEMP_PLAYBOOK=$(mktemp)
        cat > $TEMP_PLAYBOOK << EOF
---
- hosts: all
  gather_facts: true
  roles:
    - $ROLE
EOF
        PLAYBOOK_TO_RUN="$TEMP_PLAYBOOK"
      elif [ -n "$PLAYBOOK" ] && [ -f "$PLAYBOOK" ]; then
        echo "Running Ansible playbook: $PLAYBOOK"
        PLAYBOOK_TO_RUN="$PLAYBOOK"
      else
        echo "ERROR: Either PLAYBOOK or ROLE must be specified"
        echo "Available playbooks:"
        find . -name "*.yml" -type f | grep -v roles/ | grep -v requirements.yml
        echo -e "\nAvailable roles:"
        ls -la ${ANSIBLE_ROLES_PATH}/
        exit 1
      fi
      
      # Добавляем теги если указаны
      if [ -n "$TAGS" ]; then
        CMD_OPTS="$CMD_OPTS --tags '$TAGS'"
      fi
      
      # Пропускаем теги если указаны
      if [ -n "$SKIP_TAGS" ]; then
        CMD_OPTS="$CMD_OPTS --skip-tags '$SKIP_TAGS'"
      fi
      
      # Ограничение по хостам
      if [ -n "$LIMIT" ]; then
        CMD_OPTS="$CMD_OPTS --limit '$LIMIT'"
      fi
      
      # Режим проверки (dry-run)
      if [ "$CHECK_MODE" = "true" ]; then
        CMD_OPTS="$CMD_OPTS --check"
      fi
      
      # Уровень детализации
      if [ -n "$VERBOSITY" ]; then
        VERBOSE_LEVEL=""
        for i in $(seq 1 ${#VERBOSITY}); do
          VERBOSE_LEVEL="${VERBOSE_LEVEL}v"
        done
        CMD_OPTS="$CMD_OPTS -$VERBOSE_LEVEL"
      fi
      
      # Дополнительные переменные
      if [ -n "$EXTRA_VARS" ]; then
        CMD_OPTS="$CMD_OPTS --extra-vars '$EXTRA_VARS'"
      fi
      
      # Выполнение команды
      echo "Running command: $ANSIBLE_CMD $PLAYBOOK_TO_RUN $CMD_OPTS"
      eval "$ANSIBLE_CMD $PLAYBOOK_TO_RUN $CMD_OPTS"
      
      # Очистка временных файлов
      if [ -n "$TEMP_PLAYBOOK" ]; then
        rm -f $TEMP_PLAYBOOK
      fi
  rules:
    - when: manual
  allow_failure: false
```

Конфигурация для GUI (GitLab CI/CD Variables):

Создайте следующие переменные в Settings > CI/CD > Variables:

1. PLAYBOOK (optional):
   · Description: Path to playbook file (e.g., site.yml, deploy.yml)
   · Type: File или Variable
2. ROLE (optional):
   · Description: Role name to execute
   · Type: Variable
3. INVENTORY:
   · Value: production
   · Description: Inventory file name (without extension)
   · Type: Variable
4. TAGS:
   · Description: Comma-separated tags to run
   · Type: Variable
5. SKIP_TAGS:
   · Description: Comma-separated tags to skip
   · Type: Variable
6. LIMIT:
   · Description: Limit execution to specific hosts
   · Type: Variable
7. EXTRA_VARS:
   · Description: Extra variables in JSON or key=value format
   · Type: Variable
8. CHECK_MODE:
   · Value: false
   · Description: Dry-run mode (true/false)
   · Type: Variable
9. VERBOSITY:
   · Value: v
   · Description: Verbosity level (v, vv, vvv, vvvv)
   · Type: Variable

Как использовать:

1. Запуск через GitLab GUI:

· Перейдите в CI/CD > Pipelines
· Нажмите Run pipeline
· Выберите branch
· Заполните переменные:
  · Для запуска плейбука: укажите PLAYBOOK=deploy.yml
  · Для запуска роли: укажите ROLE=nginx
· Нажмите Run pipeline

2. Запуск с тегами:

```
TAGS: "deploy,webserver"
SKIP_TAGS: "restart"
```

3. Запуск с дополнительными переменными:

```
EXTRA_VARS: '{"version": "1.0.0", "environment": "staging"}'
```

4. Ограничение по хостам:

```
LIMIT: "web_servers"
```

Структура проекта:

```
├── .gitlab-ci.yml
├── ansible.cfg
├── requirements.yml
├── requirements.txt
├── site.yml
├── deploy.yml
├── inventory/
│   ├── production
│   └── staging
└── roles/
    ├── nginx/
    ├── postgresql/
    └── redis/
```

Расширенные возможности:

1. Для множественных окружений:

```yaml
deploy_to_production:
  stage: deploy
  variables:
    INVENTORY: "production"
  rules:
    - if: $CI_COMMIT_TAG
      when: manual

deploy_to_staging:
  stage: deploy
  variables:
    INVENTORY: "staging"
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"
```

2. С секретами через Vault:

```yaml
  before_script:
    - |
      if [ -n "$ANSIBLE_VAULT_PASSWORD" ]; then
        echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
        CMD_OPTS="$CMD_OPTS --vault-password-file .vault_pass"
      fi
```

3. Кэширование ролей:

```yaml
cache:
  paths:
    - ${ANSIBLE_ROLES_PATH}
  key: "${CI_PROJECT_ID}-ansible-roles"
```

Этот шаблон предоставляет гибкий механизм для запуска Ansible с возможностью выбора через GUI и поддерживает различные сценарии использования.