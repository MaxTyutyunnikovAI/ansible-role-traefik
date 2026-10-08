Ansible-роль: Traefik
=========

Ansible-роль для установки Traefik v3 в качестве systemd-сервиса: бинарный файл загружается с GitHub Releases, статическая и динамическая конфигурация описываются декларативно через переменные роли, юнит systemd усилен настройками безопасности, логи ротируются через logrotate без перезапуска сервиса.

Возможности
-----------

* Установка бинарного файла Traefik нужной версии идемпотентно: загрузка архива происходит только при отсутствии бинарника или изменении `traefik_version`.
* Создание системных группы и пользователя `traefik`, каталогов `/etc/traefik`, `/var/log/traefik`, `/var/lib/traefik`.
* Декларативная статическая конфигурация (`traefik_static_config` → YAML через `to_nice_yaml`).
* Динамическая конфигурация через File Provider (`traefik_dynamic_configs` → отдельные файлы в `conf.d/` с `watch: true`).
* Закалённый (hardened) systemd-юнит: `NoNewPrivileges`, `ProtectSystem`, `PrivateTmp`, `CAP_NET_BIND_SERVICE` и т.д.
* Перезапуск сервиса **только после** успешной валидации `traefik --check` (обработчик-цепочка `validate_and_restart_traefik`) — «битая» конфигурация никогда не поднимет сервис.
* Настройка logrotate с пересылкой `SIGUSR1` процессу Traefik (переоткрытие логов без простоя).
* Поддержка SELinux (восстановление контекста бинарника) и разных архитектур (amd64/arm64/386).

Требования
----------

* Дистрибутив на базе systemd (роль выполняет `assert` при проверке факта `service_mgr`).
* Ansible >= 2.10.
* На целевом хосте должны присутствовать `tar`, `gzip`, `procps-ng` (для unarchive и pgrep) и `logrotate` — в Molecule-сценарии они ставятся в `pre_tasks`.

Переменные роли
--------------

Все переменные можно переопределить из inventory/group_vars (см. `defaults/main.yml`). Основные:

```yaml
# Версия и загрузка
traefik_version: "v3.6.2"                                   # закреплённый тег релиза v3.x
traefik_repo_url: "https://github.com/traefik/traefik"
traefik_binary_url: ".../traefik_{{ traefik_version }}_linux_{{ go_arch }}.tar.gz"

# Пользователи и пути
traefik_service_name: "traefik"
traefik_system_user: "traefik"
traefik_system_group: "traefik"
traefik_bin_dir: "/usr/local/bin"
traefik_config_dir: "/etc/traefik"
traefik_static_config_file: "{{ traefik_config_dir }}/traefik.yml"
traefik_dynamic_config_dir: "{{ traefik_config_dir }}/conf.d"   # каталог File Provider
traefik_log_dir: "/var/log/traefik"
traefik_data_dir: "/var/lib/traefik"                            # acme.json, runtime state

# Права владельцев и режимы
traefik_config_mode: "0640"
traefik_directory_mode: "0750"

# Параметры systemd-юнита
traefik_start_timeout: 30
traefik_restart_sec: 5
traefik_limit_nofile: 1048576
traefik_extra_capabilities: []        # например ["CAP_NET_RAW"]
traefik_extra_unit_vars: {}           # произвольные дополнительные ключи [Service]

# Logrotate
traefik_logrotate_enabled: true
traefik_logrotate_frequency: "daily"
traefik_logrotate_rotate: 14
traefik_logrotate_maxsize: "100M"
```

Статическая конфигурация по умолчанию включает entry points `web` (:80 с редиректом на HTTPS) и `websecure` (:443, certResolver `letsencrypt`), JSON-логирование, accessLog, API dashboard и file provider. Полностью переопределяется словарём `traefik_static_config`.

Динамическая конфигурация задаётся списком `traefik_dynamic_configs`; каждый элемент разворачивается в отдельный файл `<name>.yml` в каталоге `conf.d`:

```yaml
traefik_dynamic_configs:
  - name: tls-options
    content:
      tls:
        options:
          default:
            minVersion: VersionTLS12
            sniStrict: true
```

Пример плейбука
---------------

Установка Traefik как systemd-сервиса:

```yaml
- hosts: servers
  become: true
  roles:
    - role: w0.traefik
      traefik_version: v3.6.2
```

Тестирование
------------

Роль содержит Molecule-сценарий (`molecule/default`) с docker-драйвером на базе RockyLinux 9: проверка синтаксиса, converge, идемпотентность, check-режим и verify.

```bash
molecule test
```

Лицензия
-------

Apache-2.0
