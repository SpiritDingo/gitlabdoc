Вот универсальный шаблон .gitlab-ci.yml для запуска нескольких Ansible playbooks с выбором через переменные CI/CD:

```yaml
stages:
  - validate
  - deploy

variables:
  ANSIBLE_FORCE_COLOR: "1"
  PLAYBOOKS_DIR: "ansible/playbooks"
  INVENTORY_FILE: "inventory/production"

# ------------------------------------------------------------
# ПРОВЕРКА КОНФИГУРАЦИИ
# ------------------------------------------------------------
validate_ansible:
  stage: validate
  image: 
    name: willhallonline/ansible:2.16-alpine
    entrypoint: [""]
  script:
    - ansible --version
    - ansible-lint --version || echo "ansible-lint not installed"
    - |
      if [ -f "$PLAYBOOKS_DIR/requirements.yml" ]; then
        ansible-galaxy install -r $PLAYBOOKS_DIR/requirements.yml
      fi
    - ansible-inventory -i $INVENTORY_FILE --graph
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $VALIDATE_ONLY == "true"
  tags:
    - ansible

# ------------------------------------------------------------
# ВЫБОР PLAYBOOK ДЛЯ ЗАПУСКА
# ------------------------------------------------------------
.deploy_template: &deploy_definition
  stage: deploy
  image:
    name: willhallonline/ansible:2.16-alpine
    entrypoint: [""]
  before_script:
    - |
      if [ -f "$PLAYBOOKS_DIR/requirements.yml" ]; then
        ansible-galaxy install -r $PLAYBOOKS_DIR/requirements.yml
      fi
  script:
    - echo "Запуск playbook: $PLAYBOOK_NAME"
    - echo "Дополнительные теги: $TAGS"
    - echo "Дополнительные переменные: $EXTRA_VARS"
    - |
      ansible-playbook \
        -i $INVENTORY_FILE \
        $PLAYBOOKS_DIR/$PLAYBOOK_NAME \
        ${TAGS:+--tags "$TAGS"} \
        ${EXTRA_VARS:+--extra-vars "$EXTRA_VARS"} \
        ${VERBOSITY:+-$VERBOSITY} \
        ${CHECK_MODE:+--check} \
        ${DIFF_MODE:+--diff}
  rules:
    - if: $PLAYBOOK_NAME
  tags:
    - ansible
  artifacts:
    reports:
      junit: reports/junit-*.xml
    paths:
      - ansible/logs/
    expire_in: 1 week

# ------------------------------------------------------------
# ОПРЕДЕЛЕНИЕ JOBS ДЛЯ КАЖДОГО PLAYBOOK
# ------------------------------------------------------------
deploy_webservers:
  <<: *deploy_definition
  variables:
    PLAYBOOK_NAME: "deploy-web.yml"
  rules:
    - if: $RUN_DEPLOY_WEB == "true"
      when: always
    - if: $PLAYBOOK_NAME == "deploy-web.yml"

deploy_database:
  <<: *deploy_definition
  variables:
    PLAYBOOK_NAME: "deploy-db.yml"
  rules:
    - if: $RUN_DEPLOY_DB == "true"
      when: always
    - if: $PLAYBOOK_NAME == "deploy-db.yml"

deploy_monitoring:
  <<: *deploy_definition
  variables:
    PLAYBOOK_NAME: "deploy-monitoring.yml"
  rules:
    - if: $RUN_DEPLOY_MONITORING == "true"
      when: always
    - if: $PLAYBOOK_NAME == "deploy-monitoring.yml"

# ------------------------------------------------------------
# РУЧНОЙ ЗАПУСК С ВЫБОРОМ PLAYBOOK
# ------------------------------------------------------------
manual_deploy:
  <<: *deploy_definition
  stage: deploy
  variables:
    PLAYBOOK_NAME: "$MANUAL_PLAYBOOK"
  script:
    - |
      if [ -z "$MANUAL_PLAYBOOK" ]; then
        echo "Ошибка: Не указан MANUAL_PLAYBOOK"
        echo "Доступные playbooks:"
        ls -la $PLAYBOOKS_DIR/*.yml | awk -F/ '{print $NF}'
        exit 1
      fi
      
      if [ ! -f "$PLAYBOOKS_DIR/$MANUAL_PLAYBOOK" ]; then
        echo "Ошибка: Playbook $MANUAL_PLAYBOOK не найден"
        exit 1
      fi
      
      ansible-playbook \
        -i $INVENTORY_FILE \
        $PLAYBOOKS_DIR/$MANUAL_PLAYBOOK \
        ${MANUAL_TAGS:+--tags "$MANUAL_TAGS"} \
        ${MANUAL_EXTRA_VARS:+--extra-vars "$MANUAL_EXTRA_VARS"} \
        ${VERBOSITY:+-$VERBOSITY} \
        ${CHECK_MODE:+--check} \
        ${DIFF_MODE:+--diff}
  rules:
    - when: manual
      allow_failure: false
  tags:
    - ansible
```

📁 Структура проекта:

```
project/
├── .gitlab-ci.yml
├── ansible/
│   ├── playbooks/
│   │   ├── deploy-web.yml
│   │   ├── deploy-db.yml
│   │   ├── deploy-monitoring.yml
│   │   └── requirements.yml
│   └── inventory/
│       └── production
└── README.md
```

🔧 Переменные CI/CD для управления:

Автоматический запуск:

```bash
# Запуск конкретного playbook
PLAYBOOK_NAME="deploy-web.yml"

# Или через флаги
RUN_DEPLOY_WEB="true"
RUN_DEPLOY_DB="false"
RUN_DEPLOY_MONITORING="true"

# Дополнительные параметры
TAGS="setup,configure"
EXTRA_VARS="version=2.0"
VERBOSITY="v"  # v, vv, vvv, vvvv
CHECK_MODE="true"  # --check
DIFF_MODE="true"   # --diff
```

Ручной запуск через UI GitLab:

1. Зайти в CI/CD → Pipelines → Run pipeline
2. Указать переменные:
   ```
   MANUAL_PLAYBOOK="deploy-web.yml"
   MANUAL_TAGS="install"
   MANUAL_EXTRA_VARS="env=staging"
   ```

🎯 Альтернативная версия с динамическим выбором:

```yaml
dynamic_deploy:
  stage: deploy
  image:
    name: willhallonline/ansible:2.16-alpine
    entrypoint: [""]
  script:
    - |
      # Выбор playbook через переменную или список
      if [ -n "$SELECTED_PLAYBOOKS" ]; then
        PLAYBOOKS_LIST=$(echo "$SELECTED_PLAYBOOKS" | tr ',' '\n')
      elif [ -n "$PLAYBOOK_NAME" ]; then
        PLAYBOOKS_LIST="$PLAYBOOK_NAME"
      else
        PLAYBOOKS_LIST=$(ls $PLAYBOOKS_DIR/*.yml | xargs -n1 basename)
      fi
      
      for playbook in $PLAYBOOKS_LIST; do
        echo "▶️ Запуск $playbook"
        ansible-playbook \
          -i $INVENTORY_FILE \
          $PLAYBOOKS_DIR/$playbook \
          ${TAGS:+--tags "$TAGS"} \
          ${EXTRA_VARS:+--extra-vars "$EXTRA_VARS"}
        
        if [ $? -ne 0 ]; then
          echo "❌ Ошибка в $playbook"
          exit 1
        fi
      done
  rules:
    - if: $SELECTED_PLAYBOOKS || $PLAYBOOK_NAME
  variables:
    SELECTED_PLAYBOOKS: "deploy-web.yml,deploy-db.yml"
```

⚙️ Пример .gitlab-ci.yml для разных окружений:

```yaml
include:
  - template: 'Workflows/Branch-Pipelines.gitlab-ci.yml'

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH == "staging"
    - if: $MANUAL_PLAYBOOK  # Всегда запускать при ручном выборе

# Окружение production
.deploy_production:
  extends: .deploy_template
  environment:
    name: production
    url: https://example.com
  variables:
    INVENTORY_FILE: "inventory/production"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Для production ручной запуск

# Окружение staging
.deploy_staging:
  extends: .deploy_template
  environment:
    name: staging
    url: https://staging.example.com
  variables:
    INVENTORY_FILE: "inventory/staging"
  rules:
    - if: $CI_COMMIT_BRANCH == "staging"
```

🔐 Использование Vault (если нужно):

```yaml
before_script:
  - |
    if [ -n "$ANSIBLE_VAULT_PASSWORD" ]; then
      echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
      chmod 600 .vault_pass
    fi
    
    ansible-playbook ... --vault-password-file .vault_pass
```

Этот шаблон предоставляет гибкость:

1. Автоматический запуск по условиям (ветки, теги, переменные)
2. Ручной запуск с выбором playbook через UI GitLab
3. Проверка конфигурации перед деплоем
4. Поддержка тегов, доп. переменных, режимов проверки
5. Артефакты и отчеты о выполнении

Настройте под свои нужды, изменив inventory, образ с Ansible и список playbooks.