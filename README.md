# Ansible Playbook

  ----------------------------- ---------------------------------------------------
  **Студент**                   Демин Илья Викторович

  **GitHub репозиторий**        https://github.com/deminilyadev-maker/Ansible.git
  ----------------------------- ---------------------------------------------------

## Описание

Playbook `site.yml` предназначен для автоматизированной установки и
настройки компонентов инфраструктуры на виртуальных машинах Yandex
Cloud.

Playbook содержит три основных play:

-   **Install ClickHouse** --- устанавливает ClickHouse, запускает
    сервис и создаёт базу данных `logs`;
-   **Install Lighthouse** --- устанавливает NGINX и зависимости,
    загружает Lighthouse из GitHub, размещает его на сервере и
    настраивает NGINX для публикации Lighthouse;
-   **Install Vector** --- загружает дистрибутив Vector заданной версии,
    распаковывает его и устанавливает Vector.

Конфигурационные файлы разворачиваются с помощью Jinja2-шаблонов. При
изменении конфигурации используются handlers для применения изменений.

### Lighthouse

Для установки Lighthouse используется официальный репозиторий:

``` text
https://github.com/VKCOM/lighthouse.git
```

Основные действия:

1.  установка необходимых зависимостей;
2.  установка NGINX;
3.  клонирование Lighthouse;
4.  размещение Lighthouse в `/opt/lighthouse`;
5.  развёртывание конфигурации NGINX;
6.  перезагрузка NGINX после изменения конфигурации.

### ClickHouse

Playbook устанавливает RPM-пакеты ClickHouse указанной версии, запускает
сервис `clickhouse-server` и создаёт базу данных:

``` text
logs
```

### Vector

Playbook загружает архив Vector указанной версии, распаковывает его в
выбранную директорию и устанавливает Vector.

------------------------------------------------------------------------

## Параметры

Основные параметры вынесены в `group_vars`.

### ClickHouse

Файл:

``` text
group_vars/clickhouse.yml
```

  Параметр                Назначение
  ----------------------- -------------------------------------------
  `clickhouse_version`    Версия ClickHouse
  `clickhouse_packages`   Список устанавливаемых пакетов ClickHouse

Пример:

``` yaml
clickhouse_version: "22.3.3.44"

clickhouse_packages:
  - clickhouse-client
  - clickhouse-server
  - clickhouse-common-static
```

### Lighthouse

Файл:

``` text
group_vars/lighthouse.yml
```

  Параметр                    Назначение
  --------------------------- ---------------------------------
  `nginx_user_name`           Пользователь NGINX
  `lighthouse_vcs`            Git-репозиторий Lighthouse
  `lighthouse_location_dir`   Директория установки Lighthouse

Пример:

``` yaml
nginx_user_name: nginx
lighthouse_vcs: "https://github.com/VKCOM/lighthouse.git"
lighthouse_location_dir: "/opt/lighthouse"
```

### Vector

Файл:

``` text
group_vars/vector.yml
```

  Параметр           Назначение
  ------------------ ---------------
  `vector_version`   Версия Vector

Пример:

``` yaml
vector_version: "0.34.1"
```

------------------------------------------------------------------------

## Inventory

Для запуска используется production inventory:

``` text
inventory/prod.yml
```

В нём определены группы:

``` text
clickhouse
lighthouse
vector
```

Каждая группа содержит соответствующую виртуальную машину.

Подключение выполняется по SSH от пользователя `idemin` с использованием
приватного ключа:

``` yaml
ansible_user: idemin
ansible_ssh_private_key_file: /home/ilya/ssh-key-1789408953122/ssh-key-1789408953122
```

Запуск playbook:

``` bash
ansible-playbook -i inventory/prod.yml site.yml
```

------------------------------------------------------------------------

## Теги

В текущей версии `site.yml` пользовательские Ansible-теги для задач **не
определены**.

Поэтому запуск отдельных задач через:

``` bash
--tags
```

не используется.

Выбор целевых узлов выполняется через inventory:

``` bash
-i inventory/prod.yml
```

Для проверки playbook используются стандартные параметры Ansible:

``` bash
--check
```

Проверка без внесения изменений.

``` bash
--diff
```

Отображение изменений в управляемых файлах.

Примеры:

``` bash
ansible-playbook -i inventory/prod.yml site.yml --check
```

``` bash
ansible-playbook -i inventory/prod.yml site.yml --diff
```

------------------------------------------------------------------------

## Структура проекта

``` text
08-ansible-03-yandex/
├── group_vars/
│   ├── clickhouse.yml
│   ├── lighthouse.yml
│   └── vector.yml
├── inventory/
│   └── prod.yml
├── templates/
│   ├── lighthouse.conf.j2
│   ├── nginx.conf.j2
│   ├── vector.service.j2
│   └── vector.yml.j2
├── screenshots/
├── site.yml
└── README.md
```
