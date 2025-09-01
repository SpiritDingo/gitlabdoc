Настройка интеграции GitLab с SonarQube позволяет автоматизировать анализ кода, проверку качества и безопасность в ваших проектах. Вот пошаговое руководство по настройке:

🔧 1. Предварительные требования

· Версии GitLab: Для самоуправляемых инсталляций GitLab требуется версия 11.7+ (рекомендуется 15.6+) .
· Издание SonarQube: Некоторые функции (например, анализ веток, декорация merge request) доступны только в Developer Edition и выше .
· Сертификаты: Если используется self-signed SSL-сертификат в GitLab, его необходимо импортировать в хранилище доверенных сертификатов Java на сервере SonarQube командой:
  ```bash
  keytool -import -alias GitLabCACert -file /path/to/cert.crt -keystore "$JAVA_HOME/lib/security/cacerts" -storepass changeit
  [citation:5].
  ```

⚙️ 2. Настройка аутентификации через GitLab OAuth

· Создайте OAuth-приложение в GitLab:
  · Redirect URI: Укажите https://<sonarqube-url>/oauth2/callback/gitlab (например, http://myIp:9000/oauth2/callback/gitlab для HTTP) .
  · Scopes: Выберите api (для синхронизации групп) или read_user (только аутентификация) .
· В SonarQube (Администрирование > Configuration > General Settings > ALM Integrations > GitLab > Authentication):
  · Введите Application ID и Secret из OAuth-приложения.
  · Включите опцию Enabled .
· Важно: Убедитесь, что Server Base URL в SonarQube (Administration > General) корректно задан, иначе аутентификация может не сработать .

🔗 3. Настройка интеграции на глобальном уровне

· В SonarQube перейдите в Administration > Configuration > General Settings > ALM Integrations > GitLab.
· Заполните параметры:
  · GitLab API URL: Например, https://gitlab.example.com/api/v4 .
  · Personal Access Token: Токен пользователя GitLab с правами api (рекомендуется использовать dedicated-аккаунт с правами Reporter) .
· Проверьте соединение. Если возникает ошибка "Could not validate GitLab url", убедитесь, что сертификат GitLab импортирован в Java .

📥 4. Импорт проектов GitLab в SonarQube

· На странице проектов SonarQube нажмите Add project > GitLab.
· Предоставьте personal access token с scope read_api для доступа к списку проектов .
· Выберите проект из списка. Настройки декорации merge request будут настроены автоматически .

🛠️ 5. Настройка анализа в GitLab CI/CD

· Добавьте переменные окружения в GitLab:
  · SONAR_TOKEN: Сгенерированный в SonarQube токен для проекта.
  · SONAR_HOST_URL: URL сервера SonarQube .
· Настройте файл .gitlab-ci.yml:
  ```yaml
  sonarqube-check:
    stage: test
    image: sonarsource/sonar-scanner-cli:latest
    variables:
      GIT_DEPTH: 0  # Обязательно для анализа веток
    script:
      - sonar-scanner
        -Dsonar.qualitygate.wait=true  # Ожидать результат Quality Gate
    rules:
      - if: $CI_COMMIT_BRANCH == "main"  # Анализ основной ветки
      - if: $CI_MERGE_REQUEST_IID  # Анализ для merge request
  [citation:2][citation:8].
  ```
· Для анализа merge request передавайте параметры:
  ```bash
  -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME
  [citation:4].
  ```

🖼️ 6. Декорация Merge Request (Developer Edition+)

· Для ручного создания проекта настройте вручную:
  · Project Settings > General Settings > Pull Request Decoration.
  · Укажите Configuration name и GitLab Project ID .
· Для монорепозиториев включите Enable mono repository support и настройте включение нужных файлов через Analysis Scope .

⚠️ 7. Решение распространенных проблем

· Ошибка аутентификации: Проверьте Server Base URL в SonarQube и redirect URI в GitLab OAuth .
· Нет комментариев в merge request: Убедитесь, что анализ запущен в режиме preview и обнаружены новые issues .
· Сканирование не запускается: Убедитесь, что переменные SONAR_TOKEN и SONAR_HOST_URL установлены в GitLab CI .

💎 Заключение

Интеграция SonarQube с GitLab автоматизирует контроль качества кода. Для полноценной работы используйте Developer Edition или выше. Для монорепозиториев требуется Enterprise Edition .

Для более сложных сценариев (например, монорепозитории) обратитесь к документации .