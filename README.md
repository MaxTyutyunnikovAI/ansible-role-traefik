Ansible-роль: Traefik
=========

Ansible-роль для установки Traefik v3 в качестве systemd-сервиса: бинарный файл загружается с GitHub Releases, статическая и динамическая конфигурация описываются декларативно через переменные роли, юнит systemd усилен настройками безопасности, логи ротируются через logrotate без перезапуска сервиса.

Возможности
-----------

* Установка бинарного файла Traefik нужной версии идемпотентно: загрузка архива происходит только при отсутствии бинарника или изменении `traefik_version`.
* Создание системных группы и пользователя `traefik`, каталогов `/etc/traefik`, `/var/log/traefik`, `/var/lib/traefik`.
* Декларативная статическая конфигурация (`traefik_static_config` → YAML через `to_nice_yaml`).
* Динамическая конфигурация через File Provider (`traefik_dynamic_configs` → отдельные файлы в `conf.d/` с `watch: true`).
* Подключение к Kubernetes/k3s: первоклассенные переменные `traefik_kubernetes_*` включают `kubernetesIngress`, `kubernetesCRD` и `kubernetesGateway` (endpoint + SA-токен + CA; в статике Traefik v3 нет поля `kubeconfig`).
* Закалённый (hardened) systemd-юнит: `NoNewPrivileges`, `ProtectSystem`, `PrivateTmp`, `CAP_NET_BIND_SERVICE` и т.д.
* Перезапуск сервиса **только после** успешной проверки новой конфигурации тайм-аутным пробным запуском (обработчик-цепочка `validate_and_restart_traefik`): конфигурация парсится, демоны переживают >= N секунд и не пишут ошибок → старт; иначе — откат на `traefik.yml.bak` и прерывание плейбока. «битая» конфигурация никогда не поднимет сервис (у Traefik v3 нет офлайн-режима `--check`, см. `THINKING.md`).
* Секреты (`traefik_envs`, например `CF_API_TOKEN`) вынесены в root-файл `traefik.env` (mode `0600`) и подключаются через `EnvironmentFile` — в world-readable юнит (0644) они не попадают.
* Ротация logrotate с пересылкой `SIGUSR1` процессу Traefik (переоткрытие логов без простоя).
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
traefik_version: "v3.7.14"                                  # закреплённый тег релиза v3.x
traefik_repo_url: "https://github.com/traefik/traefik"
traefik_binary_url: ".../traefik_{{ traefik_version }}_linux_{{ go_arch }}.tar.gz"

# Пользователи и пути
traefik_service_name: "traefik"
traefik_system_user: "traefik"
traefik_system_group: "traefik"
traefik_bin_dir: "/usr/local/bin"
traefik_config_dir: "/etc/traefik"
traefik_static_config_file: "{{ traefik_config_dir }}/traefik.yml"
traefik_config_backup_file: "{{ traefik_static_config_file }}.bak"  # откат при провале пробы
traefik_dynamic_config_dir: "{{ traefik_config_dir }}/conf.d"   # каталог File Provider
traefik_log_dir: "/var/log/traefik"
traefik_data_dir: "/var/lib/traefik"                            # acme.json, runtime state

# Права владельцев и режимы
traefik_config_mode: "0640"
traefik_directory_mode: "0750"

# Секреты демона (ACME DNS-токены и т.п.) — root-файл 0600 + EnvironmentFile
traefik_envs: {}              # например {CF_API_TOKEN: "{{ vault_cf_token }}"}
traefik_env_file: "{{ traefik_config_dir }}/traefik.env"

# Параметры пробной валидации конфигурации (см. handlers/main.yml)
traefik_validate_timeout: 5        # секунд, которые пробный запуск должен пережить
traefik_validate_kill_after: 10    # доп. отсрочка до SIGKILL

# Параметры systemd-юнита
traefik_start_timeout: 30
traefik_restart_sec: 5
traefik_limit_nofile: 1048576
traefik_extra_capabilities: []        # например ["CAP_NET_RAW"]
traefik_extra_unit_vars: {}           # произвольные дополнительные ключи [Service]

# Плагины Traefik v3 (trial-механизм + Traefik Pilot)
traefik_pilot_enabled: false          # true -> блоки pilot/experimental.plugins + кэш плагинов
traefik_pilot_token: ""               # TRIAL-токен с pilot.traefik.io (хранить в ansible-vault)
traefik_plugins: []                   # [{name, version, moduleName}] — предзагрузка `traefik trial --download`
traefik_plugins_dir: "/etc/traefik/plugins-trial"   # кэш исходников Yaegi-плагинов

# Kubernetes-провайдеры (k3s и др.) — см. docs/examples/playbook-k3s-provider.yml
traefik_kubernetes_enabled: false      # true -> все три провайдера в статике (см. *_enabled ниже)
traefik_kubernetes_ingress_enabled: true
traefik_kubernetes_crd_enabled: true
traefik_kubernetes_gateway_enabled: true   # требует Gateway API CRDs в кластере
traefik_kubernetes_endpoint: ""            # https://127.0.0.1:6443 (API k3s)
traefik_kubernetes_token: ""               # значение или путь к файлу (v3.7+); секрет → no_log
traefik_kubernetes_cert_auth_file: ""      # CA-бандл (certAuthFilePath)
traefik_kubernetes_ingress_class: "traefik"   # "" -> без фильтра по классу
traefik_kubernetes_ingress_extra: {}       # опции только kubernetesIngress
traefik_kubernetes_crd_extra: {}           # только kubernetesCRD (allowExternalNameServices и т.п.)
traefik_kubernetes_gateway_extra: {}       # только kubernetesGateway

# Logrotate
traefik_logrotate_enabled: true
traefik_logrotate_frequency: "daily"
traefik_logrotate_rotate: 14
traefik_logrotate_maxsize: "100M"
```

Статическая конфигурация по умолчанию включает entry points `web` (:80 с редиректом на HTTPS) и `websecure` (:443, certResolver `letsencrypt`), JSON-логирование, accessLog, API dashboard и file provider. Полностью переопределяется словарём `traefik_static_config`.

> ⚠️ При переиспользовании дефолтов в собственном определении `traefik_static_config`
> (например, `{{ traefik_static_config | combine(...) }}` или
> `{{ traefik_static_config.providers | combine(...) }}` в `roles:`/`vars:`) возникает
> цикл резолва переменных Ansible — задача падает с
> `AnsibleError: ... maximum recursion depth exceeded while calling a Python object`.
> База для merge вынесена в нешаблонные plain-копии из `vars/main.yml`:
> `traefik_role_defaults.traefik_static_config` (вся статика) и
> `traefik_default_providers` (блок providers). Примеры безопасного merge —
> в [`docs/examples/`](docs/examples/).

### Ошибка `maximum recursion depth exceeded` в задаче «Generate static Traefik configuration»

Если в inventory/group_vars или в блоке `roles:` встречается переопределение,
ссылающееся на саму определяемую переменную:

```yaml
# НЕПРАВИЛЬНО — цикл резолва (плюс docker-опции попадают в корень providers:)
traefik_static_config: >-
  {{ traefik_static_config
     | combine({'providers':
                  traefik_static_config.providers
                  | combine(docker_provider_opts)}, recursive=True) }}
```

то задача `Generate static Traefik configuration from traefik_static_config dict`
падает с `AnsibleError: ... maximum recursion depth exceeded while calling a
Python object`. Исправление — mergить в plain-копию дефолтов роли и вкладывать
опции провайдера под его ключом:

```yaml
# ПРАВИЛЬНО
traefik_static_config: >-
  {{ traefik_role_defaults.traefik_static_config
     | combine({'providers': {'docker': docker_provider_opts}},
               recursive=True) }}
```

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

Кроме декларативного списка роль умеет **генерировать динамические конфиги из
собственного Jinja2-шаблона** — когда ресурсы нужно вычислить динамически (по
фактам хоста, из словаря inventory, циклом и т.п.):

```yaml
# путь разрешается относительно templates/ роли (или абсолютный на контроллере)
traefik_dynamic_configs_template: dynamic-routers.yml.j2
traefik_dynamic_configs_template_name: 90-generated   # -> conf.d/90-generated.yml
traefik_dynamic_configs_file_extension: ".yml"

# пример данных, на которых работает идущий в роли шаблон-образец:
traefik_generated_routers:
  app:
    host: app.example.com
    url: http://127.0.0.1:8080
    middlewares: [security-headers, https-redirect]
```

Внутри шаблона доступны все переменные роли, `ansible_facts` и сам список
`traefik_dynamic_configs`. Файл рендерится идемпотентно и, как и остальные
конфиги, уведомляет цепочку обработчиков валидации; при сбросе опции
(`traefik_dynamic_configs_template: ""`, значение по умолчанию) сгенерированный
файл аккуратно удаляется. Готовый образец — `templates/dynamic-routers.yml.j2`
(http.services/http.routers из словаря `traefik_generated_routers`).

Пример плейбука
---------------

Установка Traefik как systemd-сервиса:

```yaml
- hosts: servers
  become: true
  roles:
    - role: w0.traefik
      traefik_version: v3.7.14
```

Готовые примеры подключения провайдеров (Docker, Kubernetes/k3s) и установки
плагинов — в каталоге [`docs/`](docs/README.md):

* `docs/examples/playbook-docker-provider.yml` — Docker Provider + LABEL'ы контейнеров + плагин (`experimental.plugins`, `traefik trial --download`);
* `docs/examples/playbook-k3s-provider.yml` — Kubernetes-провайдеры (Ingress/CRD/Gateway) на базе k3s через переменные `traefik_kubernetes_*` + CRD-Middleware на плагине.

Тестирование
------------

Роль содержит Molecule-сценарий (`molecule/default`) с docker-драйвером на базе RockyLinux 9: проверка синтаксиса, converge, идемпотентность и verify (версия бинарника, откатный бэкап конфигурации, работающие entry points, редирект HTTP→HTTPS, TLS, закалка юнита).

```bash
molecule test
```

Лицензия
-------

Apache-2.0
