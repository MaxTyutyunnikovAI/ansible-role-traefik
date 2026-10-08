# Журнал изменений (Changelog)

Все значимые изменения в этом проекте документируются в этом файле.
Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.1.0/),
проект придерживается [семантического версионирования](https://semver.org/lang/ru/).

## [Не выпущено]

### Добавлено
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
