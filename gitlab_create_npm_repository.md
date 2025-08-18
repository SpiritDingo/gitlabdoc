Создание NPM Registry в GitLab

GitLab предоставляет встроенный реестр пакетов NPM, который можно использовать для хранения и управления вашими приватными пакетами. Вот как его настроить:

1. Включение NPM Registry в GitLab

1. Убедитесь, что в вашем GitLab включена функция Package Registry:
   · Для GitLab.com она включена по умолчанию
   · Для self-hosted GitLab проверьте настройки в Admin Area > Settings > Packages and registries

2. Настройка проекта

Каждый проект GitLab автоматически получает свой собственный NPM Registry. Для доступа к нему:

1. Перейдите в ваш проект в GitLab
2. В левом меню выберите Deploy > Package Registry

3. Настройка .npmrc

Для аутентификации и использования реестра создайте/измените файл .npmrc:

```ini
@your-scope:registry=https://gitlab.example.com/api/v4/packages/npm/
//gitlab.example.com/api/v4/packages/npm/:_authToken=${CI_JOB_TOKEN}
```

Замените:

· your-scope - ваша область именования пакетов (например, название компании)
· gitlab.example.com - адрес вашего GitLab (для GitLab.com используйте gitlab.com)

4. Аутентификация

Для публикации пакетов вам понадобится токен доступа:

· Личный токен (для ручной публикации)
· CI_JOB_TOKEN (для публикации из CI/CD)

5. Публикация пакета

1. Убедитесь, что в package.json указаны:
   ```json
   {
     "name": "@your-scope/package-name",
     "version": "1.0.0",
     "publishConfig": {
       "@your-scope:registry": "https://gitlab.example.com/api/v4/packages/npm/"
     }
   }
   ```
2. Выполните команду:
   ```bash
   npm publish
   ```

6. Установка пакета

Для установки пакета из вашего реестра:

```bash
npm install @your-scope/package-name
```

Дополнительные настройки для self-hosted GitLab

Если вы используете собственный GitLab, убедитесь, что:

1. Включен Package Registry в конфигурации GitLab (gitlab.rb):
   ```ruby
   gitlab_rails['packages_enabled'] = true
   gitlab_rails['packages_storage_path'] = "/var/opt/gitlab/gitlab-rails/shared/packages"
   ```
2. При необходимости настройте объектное хранилище для пакетов
3. Перезапустите GitLab после изменения конфигурации

Теперь у вас есть приватный NPM Registry в GitLab для управления вашими пакетами!