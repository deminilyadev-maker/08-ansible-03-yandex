# Домашнее задание «Ansible. Playbook»

----------------------------------- ---------------------------------------------------
**Студент**                         Демин Илья Викторович

**GitHub репозиторий**              https://github.com/deminilyadev-maker/Ansible.git
----------------------------------- ---------------------------------------------------

## Оглавление

- [Задание 1](#задание-1)
- [Задание 2](#задание-2)
- [Задание 3](#задание-3)
- [Задание 4](#задание-4)
- [Задание 5](#задание-5)
- [Задание 6](#задание-6)
- [Задание 7](#задание-7)
- [Задание 8](#задание-8)
- [Итог](#итог)

------------------------------------------------------------------------

# Задание 1

## Подготовка inventory `prod.yml`

Для выполнения задания был подготовлен inventory-файл:

```text
inventory/
└── prod.yml
```

В inventory определены группы `clickhouse` и `vector`. Для учебного окружения обе группы используют тестовый контейнер `centos`.

```yaml
clickhouse:
  hosts:
    centos:

vector:
  hosts:
    centos:
```

Проверка структуры inventory:

```bash
ansible-inventory -i inventory/prod.yml --graph
```

Проверка доступности группы Vector:

```bash
ansible -i inventory/prod.yml vector -m ping
```

Результат:

```text
centos | SUCCESS
```

Для Vector создана переменная версии:

```text
group_vars/
└── vector/
    └── vector.yml
```

```yaml
vector_version: "0.34.1"
```

Проверка:

```bash
ansible-inventory -i inventory/prod.yml --host centos
```

В выводе присутствует `vector_version: 0.34.1`.

### Скриншот

![Inventory prod](screenshots/Task1_inventory_prod.png)

------------------------------------------------------------------------

# Задание 2

## Установка и настройка ClickHouse и Vector

В `site.yml` реализованы два play:

```text
site.yml
├── Install Clickhouse
└── Install Vector
```

### ClickHouse

Используется версия:

```yaml
clickhouse_version: "22.3.3.44"
```

Дистрибутивы скачиваются при помощи `get_url`, после чего устанавливаются через `yum`.

```yaml
- name: Install clickhouse packages
  become: true
  ansible.builtin.yum:
    name:
      - clickhouse-common-static-{{ clickhouse_version }}.rpm
      - clickhouse-client-{{ clickhouse_version }}.rpm
      - clickhouse-server-{{ clickhouse_version }}.rpm
    disable_gpg_check: true
  notify: Start clickhouse service
```

После установки вызывается handler:

```yaml
handlers:
  - name: Start clickhouse service
    become: true
    ansible.builtin.service:
      name: clickhouse-server
      state: restarted
```

После запуска создаётся база:

```sql
CREATE DATABASE logs;
```

### Vector

Для Vector используется отдельный play:

```yaml
- name: Install Vector
  hosts: vector
  become: true
```

Дистрибутив скачивается через `get_url`, распаковывается через `unarchive`, необходимые каталоги создаются через `file`.

Конфигурация должна разворачиваться через:

```text
templates/vector.yaml.j2
```

Задача конфигурации:

```yaml
- name: Deploy Vector config
  ansible.builtin.template:
    src: vector.yaml.j2
    dest: /etc/vector/vector.yaml
    mode: "0644"
  notify: Restart Vector
```

Handler:

```yaml
handlers:
  - name: Restart Vector
    ansible.builtin.service:
      name: vector
      state: restarted
```

Таким образом, при изменении конфигурации Vector автоматически перезапускается.

### Скриншот

![Установка и настройка Vector](screenshots/Task2_vector_install.png)

------------------------------------------------------------------------

# Задание 3

## Использование рекомендованных модулей

При создании tasks использовались модули, указанные в задании.

### `get_url`

Используется для скачивания дистрибутива Vector:

```yaml
ansible.builtin.get_url:
```

### `unarchive`

Используется для распаковки скачанного архива:

```yaml
ansible.builtin.unarchive:
```

Так как архив уже находится на целевом хосте, используется:

```yaml
remote_src: true
```

### `file`

Используется для создания каталогов:

```yaml
- name: Create Vector directory
  ansible.builtin.file:
    path: /opt/vector
    state: directory
    mode: "0755"
```

### `template`

Используется для развёртывания конфигурации Vector из Jinja2-шаблона:

```yaml
ansible.builtin.template:
```

### Скриншот

![Модули Ansible](screenshots/Task3_ansible_modules.png)

------------------------------------------------------------------------

# Задание 4

## Скачивание, распаковка и установка Vector

Используется версия:

```yaml
vector_version: "0.34.1"
```

Архив скачивается в:

```text
/tmp/vector-0.34.1.tar.gz
```

После распаковки была проверена структура:

```text
/opt/vector/vector-x86_64-unknown-linux-musl/
├── bin/
│   └── vector
├── config/
│   └── vector.yaml
└── ...
```

Проверка содержимого:

```bash
sudo docker exec centos find /opt/vector -maxdepth 3 -type f
```

Проверка версии:

```bash
sudo docker exec centos /opt/vector/vector-x86_64-unknown-linux-musl/bin/vector --version
```

Официальная документация Vector подтверждает установку Linux x86_64 из архива `x86_64-unknown-linux-musl.tar.gz`; архив содержит бинарник и конфигурационные файлы, а также service-файл для systemd. citeturn0search0

### Скриншот

![Распаковка Vector](screenshots/Task4_vector_unarchive.png)

------------------------------------------------------------------------

# Задание 5

## Проверка `ansible-lint`

Для проверки playbook используется:

```bash
ansible-lint site.yml
```

Перед этим выполняется проверка синтаксиса:

```bash
ansible-playbook -i inventory/prod.yml site.yml --syntax-check
```

Ожидаемый результат:

```text
playbook: site.yml
```

Все ошибки `ansible-lint`, обнаруженные в процессе проверки, исправляются до завершения задания.

### Скриншот

![Ansible lint](screenshots/Task5_ansible_lint.png)

------------------------------------------------------------------------

# Задание 6

## Запуск playbook с флагом `--check`

Для проверки режима dry-run используется:

```bash
ansible-playbook -i inventory/prod.yml site.yml --check
```

Режим `--check` позволяет определить предполагаемые изменения без фактического изменения состояния системы.

При этом необходимо учитывать, что `get_url` в check mode выполняет проверку URL, но не скачивает полный файл. Поэтому задачи, которые используют скачанный архив, могут отличаться по поведению от обычного запуска. citeturn0search4

### Скриншот

![Check mode](screenshots/Task6_check_mode.png)

------------------------------------------------------------------------

# Задание 7

## Проверка изменений с помощью `--diff`

После выполнения playbook используется:

```bash
ansible-playbook -i inventory/prod.yml site.yml --diff
```

Флаг `--diff` позволяет увидеть изменения файлов, в том числе конфигурации Vector, развёрнутой через `template`.

При изменении `vector.yaml.j2` задача `template` сообщает об изменении и вызывает:

```yaml
notify: Restart Vector
```

Handlers выполняются только после того, как уведомившая их задача действительно изменила состояние. citeturn1search0

### Скриншот

![Ansible diff](screenshots/Task7_diff.png)

------------------------------------------------------------------------

# Задание 8

## Повторный запуск и проверка идемпотентности

После первого применения конфигурации playbook запускается повторно:

```bash
ansible-playbook -i inventory/prod.yml site.yml --diff
```

При отсутствии изменений повторный запуск не должен изменять конфигурационные файлы и не должен приводить к ненужному перезапуску Vector.

Особенно проверяется связка:

```yaml
ansible.builtin.template:
  src: vector.yaml.j2
  dest: /etc/vector/vector.yaml
notify: Restart Vector
```

Если содержимое шаблона не изменилось, `template` не сообщает `changed`, поэтому handler не запускается.

Это подтверждает идемпотентность playbook. Ansible-модули рассчитаны на идемпотентное применение состояния и могут уведомлять handlers только при наличии изменений. citeturn1search1

### Скриншот

![Повторный запуск](screenshots/Task8_idempotent.png)

------------------------------------------------------------------------

# Итог

В ходе выполнения домашнего задания были изучены и применены:

- подготовка inventory `prod.yml`;
- работа с группами `clickhouse` и `vector`;
- переменные `group_vars`;
- установка ClickHouse;
- установка Vector из дистрибутива;
- скачивание файлов с помощью `get_url`;
- распаковка архивов с помощью `unarchive`;
- создание каталогов с помощью `file`;
- Jinja2-шаблоны;
- развёртывание конфигурации Vector через `template`;
- handlers;
- перезапуск Vector при изменении конфигурации;
- проверка синтаксиса playbook;
- проверка с помощью `ansible-lint`;
- запуск в режиме `--check`;
- анализ изменений через `--diff`;
- проверка идемпотентности повторным запуском.

Структура проекта:

```text
playbook/
├── group_vars/
│   ├── clickhouse/
│   │   └── vars.yml
│   └── vector/
│       └── vector.yml
├── inventory/
│   └── prod.yml
├── templates/
│   └── vector.yaml.j2
├── site.yml
└── README.md
```

Скриншоты результатов хранятся отдельно:

```text
screenshots/
```

и подключаются к README относительными путями.

------------------------------------------------------------------------
