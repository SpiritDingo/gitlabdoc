# Сборка DEB и RPM пакетов в GitLab CI

Вот пример конфигурации `.gitlab-ci.yml` для сборки DEB и RPM пакетов из исходников:

```yaml
stages:
  - build
  - package

variables:
  # Общие переменные
  BUILD_DIR: "build"
  PACKAGE_DIR: "packages"
  VERSION: "1.0.0"
  RELEASE: "1"

# Сборка проекта
build:
  stage: build
  script:
    - mkdir -p ${BUILD_DIR}
    - cd ${BUILD_DIR}
    - cmake ..  # или другие команды сборки вашего проекта
    - make
  artifacts:
    paths:
      - ${BUILD_DIR}/

# Сборка DEB пакета
package_deb:
  stage: package
  image: ubuntu:latest
  needs: ["build"]
  before_script:
    - apt-get update -y
    - apt-get install -y devscripts debhelper dh-make build-essential
  script:
    - mkdir -p ${PACKAGE_DIR}
    - cd ${PACKAGE_DIR}
    # Создаем структуру DEB пакета
    - dh_make --createorig --single --yes -p ${CI_PROJECT_NAME}_${VERSION}
    - cd debian
    - rm *.ex *.EX  # Удаляем примеры файлов
    # Копируем подготовленные файлы контроля (можно добавить их в репозиторий)
    - cp ${CI_PROJECT_DIR}/debian/* .
    # Собираем пакет
    - dpkg-buildpackage -us -uc
    - mv ../${CI_PROJECT_NAME}_${VERSION}-${RELEASE}_*.deb ${CI_PROJECT_DIR}/
  artifacts:
    paths:
      - "*.deb"

# Сборка RPM пакета
package_rpm:
  stage: package
  image: centos:latest
  needs: ["build"]
  before_script:
    - yum install -y rpm-build rpmdevtools make gcc
  script:
    - mkdir -p ${PACKAGE_DIR}
    - cd ${PACKAGE_DIR}
    # Устанавливаем окружение для сборки RPM
    - rpmdev-setuptree
    # Копируем исходники в SOURCES
    - cp -r ${CI_PROJECT_DIR}/${BUILD_DIR}/* ~/rpmbuild/SOURCES/
    # Копируем .spec файл (должен быть в репозитории)
    - cp ${CI_PROJECT_DIR}/rpm/${CI_PROJECT_NAME}.spec ~/rpmbuild/SPECS/
    # Собираем RPM
    - rpmbuild -bb ~/rpmbuild/SPECS/${CI_PROJECT_NAME}.spec
    - mv ~/rpmbuild/RPMS/*/${CI_PROJECT_NAME}-${VERSION}-${RELEASE}.*.rpm ${CI_PROJECT_DIR}/
  artifacts:
    paths:
      - "*.rpm"
```

## Необходимые дополнительные файлы

Для работы этого pipeline вам понадобятся:

1. **Для DEB пакета**:
   - Каталог `debian/` в корне репозитория с файлами:
     - `control` - информация о пакете
     - `rules` - правила сборки
     - `changelog` - история изменений
     - `compat` - уровень совместимости
     - `install` - файлы для установки

2. **Для RPM пакета**:
   - Каталог `rpm/` с файлом `.spec` содержащим:
     - Описание пакета
     - Зависимости
     - Инструкции сборки и установки

## Альтернативные методы

Если ваш проект использует CMake, можно упростить сборку пакетов с помощью CPack:

```yaml
package_cpack:
  stage: package
  script:
    - mkdir -p build && cd build
    - cmake ..
    - make
    - cpack -G DEB  # или -G RPM
  artifacts:
    paths:
      - "build/*.deb"
      - "build/*.rpm"
```

Для этого в CMakeLists.txt нужно добавить:
```cmake
include(InstallRequiredSystemLibraries)
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_PACKAGE_VERSION_MAJOR "1")
set(CPACK_PACKAGE_VERSION_MINOR "0")
set(CPACK_PACKAGE_VERSION_PATCH "0")
include(CPack)
```

## Советы

1. Используйте разные образы Docker для сборки DEB (Ubuntu/Debian) и RPM (CentOS/Fedora) пакетов
2. Кэшируйте зависимости для ускорения сборки
3. Тестируйте пакеты после сборки (можно добавить stage test)
4. Публикуйте пакеты в репозитории (можно добавить stage deploy)

Настройки могут варьироваться в зависимости от вашего проекта и требований к пакетам.