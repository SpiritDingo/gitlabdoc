Установка SonarScanner на Linux — это ключевой этап для интеграции статического анализа кода в процесс разработки. Вот подробное руководство по установке и настройке SonarScanner на Linux-системах.

📥 1. Загрузка и распаковка SonarScanner

· Скачайте последнюю версию SonarScanner CLI с официального сайта. Например, используйте wget:
  ```bash
  wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.5.0.2216-linux.zip
  ```
· Распакуйте архив в директорию /opt:
  ```bash
  sudo unzip sonar-scanner-cli-4.5.0.2216-linux.zip -d /opt/
  ```
· Переименуйте директорию для удобства (опционально):
  ```bash
  sudo mv /opt/sonar-scanner-4.5.0.2216-linux /opt/sonar-scanner
  ```
· Источник: .

⚙️ 2. Настройка переменных окружения

· Добавьте путь к бинарным файлам SonarScanner в переменную PATH. Для этого отредактируйте файл ~/.bashrc или /etc/profile.d/sonar-scanner.sh:
  ```bash
  echo 'export PATH=$PATH:/opt/sonar-scanner/bin' | sudo tee -a /etc/profile.d/sonar-scanner.sh
  ```
· Примените изменения:
  ```bash
  source /etc/profile.d/sonar-scanner.sh
  ```
· Проверьте установку:
  ```bash
  sonar-scanner -v
  ```
· Источник: .

🔧 3. Конфигурация SonarScanner

· Отредактируйте файл настроек /opt/sonar-scanner/conf/sonar-scanner.properties:
  ```bash
  sudo nano /opt/sonar-scanner/conf/sonar-scanner.properties
  ```
· Укажите URL вашего сервера SonarQube и токен аутентификации:
  ```
  sonar.host.url=http://localhost:9000
  sonar.token=your_authentication_token
  ```
· Если используется самоподписанный SSL-сертификат, настройте доверенные сертификаты Java .
· Источник: .

🗃️ 4. Настройка кэширования (опционально)

· Для ускорения последующих сканирований настройте кэширование файлов сканера. Используйте переменную SONAR_USER_HOME для указания пути к кэшу:
  ```bash
  export SONAR_USER_HOME=/path/to/cache/directory
  ```
· Убедитесь, что пользователь имеет права на запись в эту директорию .

🚀 5. Запуск сканирования проекта

· Перейдите в директорию вашего проекта и создайте файл sonar-project.properties с минимальными настройками:
  ```properties
  sonar.projectKey=my_project
  sonar.sources=.
  ```
· Запустите сканирование:
  ```bash
  sonar-scanner
  ```
· Для передачи параметров через командную строку:
  ```bash
  sonar-scanner -Dsonar.projectKey=myproject -Dsonar.sources=src
  ```
· Источник: .

🔍 6. Просмотр результатов

· После завершения сканирования результаты будут доступны на сервере SonarQube по URL, указанному в настройках (по умолчанию http://localhost:9000) .

🛠️ 7. Дополнительные настройки

· Память Java: Если возникают ошибки нехватки памяти, увеличьте лимит через переменную окружения:
  ```bash
  export SONAR_SCANNER_OPTS="-Xmx2G"
  ```
· Права доступа: Убедитесь, что пользователь имеет права на чтение/запись в директории проекта и кэша .
· Интеграция с CI: Для использования в CI/CD настройте переменные окружения (например, SONAR_TOKEN) в вашей системе .

❌ 8. Устранение распространенных проблем

· Ошибки памяти: Увеличьте значение -Xmx в SONAR_SCANNER_OPTS .
· Проблемы с сетью: Проверьте настройки прокси и SSL, если сервер SonarQube недоступен .
· Права доступа: Убедитесь, что пользователь имеет права на выполнение файлов в /opt/sonar-scanner/bin/ .

📊 Сводная таблица шагов установки

Шаг Действие Команда / Пример
1 Загрузка wget <URL_сканера>
2 Распаковка sudo unzip ... -d /opt/
3 Настройка PATH export PATH=$PATH:/opt/sonar-scanner/bin
4 Конфигурация sonar.host.url=http://localhost:9000
5 Запуск sonar-scanner

💎 Заключение

Установка SonarScanner на Linux требует загрузки пакета, настройки путей и параметров подключения к серверу SonarQube. Правильная конфигурация обеспечивает интеграцию статического анализа в процесс разработки, что помогает поддерживать качество кода. Для нестандартных сценариев (например, использование Docker или специфичные сети) обратитесь к официальной документации .