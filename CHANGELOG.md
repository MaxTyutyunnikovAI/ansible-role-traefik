# Журнал изменений (Changelog)

Все значимые изменения в этом проекте документируются в этом файле.
Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.1.0/),
проект придерживается [семантического версионирования](https://semver.org/lang/ru/).

## [Не выпущено]

### Исправлено
- Рекурсия шаблонов (`AnsibleError: ... maximum recursion depth exceeded while
  calling a Python object`) при подключении docker/kubernetes-провайдера:
  документированные примеры merge вида
  `{{ traefik_static_config | combine({'providers': traefik_static_config.providers | combine(...)}) }}`
  ссылались на собственное определение переменной, из-за чего Ansible зацикливал
  резолв. Примеры в `docs/` переведены на нешаблонную базу
  `traefik_role_defaults.traefik_static_config` (плюс новый plain-блок
  `traefik_default_providers` в `vars/main.yml`); предупреждение добавлено в
  комментарии `defaults/main.yml`, README и docs/README. В README и docs/README
  отдельно разобран «исправленный наполовину» вариант
  `{{ traefik_static_config | combine({'providers': traefik_static_config.providers
  | combine(docker_provider_opts)}, recursive=True) }}` — он по-прежнему
  ссылается на собственное определение (двойной цикл резолва) и одновременно
  теряет ключ провайдера; рядом приведён корректный merge.
- Опции провайдера в примерах теперь вкладываются в `providers.docker:` /
  `providers.kubernetesprovider:` вместо слияния прямо в корень секции
  `providers:` (старый вариант терял ключ провайдера в итоговом traefik.yml).

### Добавлено
- **Генерация динамических конфигов из Jinja2-шаблона**: новые переменные
  `traefik_dynamic_configs_template` (путь к шаблону, по умолчанию `""` — выключено),
  `traefik_dynamic_configs_template_name` и `traefik_dynamic_configs_file_extension`;
  `tasks/configure.yml` рендерит `conf.d/<name>.yml` идемпотентно, уведомляет цепочку
  обработчиков валидации и удаляет сгенерированный файл при отключении опции.
  Декларативный список `traefik_dynamic_configs` теперь рендерится через новый
  `templates/dynamic-config.yml.j2`; образец шаблона — `templates/dynamic-routers.yml.j2`
  (http.services/http.routers из словаря `traefik_generated_routers`). Документация —
  в README («Динамическая конфигурация»).
- Установка **плагинов Traefik v3** (trial-механизм + Traefik Pilot): переменные
  `traefik_pilot_enabled`, `traefik_pilot_token` (секрет — ansible-vault),
  `traefik_plugins` (`[{name, version, moduleName}]`), `traefik_plugins_dir`,
  `traefik_pilot_extra_props`; шаблон статики автоматически добавляет блоки
  `pilot:` и `experimental.plugins:`, а `tasks/configure.yml` идемпотентно
  предзагружает исходники плагинов через `traefik trial --download` в кэш
  `traefik_plugins_dir` (повторная загрузка — только при отсутствии кэша или смене версии).
  Закалённый юнит получает `ReadWritePaths={{ traefik_plugins_dir }}` при включённом pilot.
  Примеры подключения плагинов — в обоих плейбуках `docs/examples/`.
- Полная переработка роли под Traefik **v3** с декларативной конфигурацией.
- Переменная `traefik_version` с закреплением релиза (по умолчанию `v3.6.2`) и идемпотентная
  установка бинарного файла: архив с GitHub Releases загружается и распаковывается только при
  отсутствии бинарника или несовпадении установленной версии с запрошенной.
- Поддержка архитектур `amd64`, `arm64`, `386` через карту `go_arch_map` (`vars/main.yml`).
- Системные группа и пользователь `traefik` (без интерактивного входа, `nologin`).
- Каталоги `/etc/traefik`, `/etc/traefik/conf.d`, `/var/log/traefik`, `/var/lib/traefik`
  с настраиваемыми владельцами и режимами (`0640` / `0750` по умолчанию).
- Декларативная статическая конфигурация: словарь `traefik_static_config` рендерится в
  `traefik.yml` фильтром `to_nice_yaml(indent=2)`; по умолчанию настроены entry points
  `web` (:80 → редирект на HTTPS) и `websecure` (:443, letsencrypt), JSON-логи, accessLog,
  API dashboard и file provider.
- Динамическая конфигурация через File Provider: список `traefik_dynamic_configs` разворачивается
  в отдельные файлы `<name>.yml` в каталоге `conf.d`; по умолчанию поставляются безопасные
  TLS-опции (минимум TLS 1.2, `sniStrict`, современные шифры).
- Закалённый systemd-юнит (`templates/traefik.service.j2`): `NoNewPrivileges`, `ProtectSystem=full`,
  `ProtectHome`, `PrivateTmp`, `PrivateDevices`, `ProtectKernelTunables/Modules`,
  `RestrictSUIDSGID/Namespaces`, `LockPersonality`, `SystemCallArchitectures=native`,
  лимиты `LimitNOFILE`, `StartLimitBurst`, возможности `CAP_NET_BIND_SERVICE` для привилегированных
  портов от непривилегированного пользователя; переменные `traefik_extra_capabilities` и
  `traefik_extra_unit_vars` для тонкой настройки.
- Цепочка обработчиков `validate_and_restart_traefik`: перед любым перезапуском выполняется
  `traefik --configFile=... --check=true`; при ошибке валидации плей прерывается и сервис
  **никогда не перезапускается с некорректной конфигурацией**.
- Настройка logrotate (`/etc/logrotate.d/traefik`) с postrotate-сигналом `SIGUSR1` — Traefik
  переоткрывает лог-файлы без простоя и без перезапуска; параметры частоты, глубины ротации и
  максимального размера вынесены в переменные; ротацию можно отключить (`traefik_logrotate_enabled: false`).
- PID-файл сервиса в `/run/traefik/traefik.pid` (ExecStartPost + PIDFile) для точечной отправки SIGUSR1.
- Восстановление SELinux-контекста для бинарного файла Traefik при включённом SELinux.
- Molecule-сценарий (`molecule/default`): docker-драйвер, RockyLinux 9, последовательность
  syntax → converge → idempotence → check → verify.
- CI-воркфлоу публикации роли на Ansible Galaxy (`.github/workflows/release.yml`).

### Изменено
- Структура задач разбита на модули: `install.yml` → `configure.yml` → `systemd.yml` → `logrotate.yml`
  (входят в `tasks/main.yml` через `include_tasks`).
- Валидация конфигурации убрана из inline-параметра `validate:` шаблона: `traefik --check`
  занимает сокеты entry points, поэтому единственной авторитетной точкой проверки выбрана
  цепочка обработчиков перед перезапуском.
- Обновлены формулировки в README.md (переведены на русский, дополнены описанием переменных).

## [0.1.x] — исторические базовые версии
- Начальная версия роли: установка Traefik как systemd-сервиса с минимальным набором переменных
  (`traefik_version`, `traefik_service_name`, пути и учётная запись).
