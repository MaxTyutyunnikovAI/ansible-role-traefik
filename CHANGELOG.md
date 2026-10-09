# Журнал изменений (Changelog)

Все значимые изменения в этом проекте документируются в этом файле.
Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.1.0/),
проект придерживается [семантического версионирования](https://semver.org/lang/ru/).

## [Не выпущено]

### Добавлено
- **Первоклассная поддержка Kubernetes/k3s**: переменные `traefik_kubernetes_*`
  (`traefik_kubernetes_enabled` + по-провайдерные `ingress/crd/gateway_enabled`,
  connection `endpoint`/`token`/`cert_auth_file`, `ingress_class`, per-provider
  `ingress/crd/gateway_extra`) рендерятся шаблоном статики в блоки
  `providers.kubernetesIngress/CRD/Gateway` через рекурсивный `combine` —
  file-провайдер и вручную влитые провайдеры (docker и т.п.) сохраняются.
  Опции разделены по провайдерам намеренно: Traefik парсит статику строго, и
  чужой ключ (например `allowExternalNameServices` вне `kubernetesCRD`) валит
  демон с `field not found`. В статике Traefik v3 **нет поля `kubeconfig`**
  (проверено на v3.7.14) — внешний клиент задаётся только
  endpoint+token+certAuthFilePath (альтернатива — env `KUBECONFIG` через
  `traefik_envs`); ошибки инициализации провайдеров идут в лог-файл и не видны
  в stderr, поэтому не влияют на тайм-аутную пробу перезапуска. Токен
  попадает в статику → задача шаблона автоматически идёт с `no_log` при
  непустом `traefik_kubernetes_token`. Пример
  `docs/examples/playbook-k3s-provider.yml` переписан под новые переменные
  (старый использовал несуществующий ключ `kubernetesprovider` и невалидный
  kubeconfig), README дополнен описанием переменных.

### Исправлено
- **Цепочка валидации перезаписана под Traefik v3**: офлайн-режима `--check` в v3 не
  существует (без `--configFile` — ошибка `field not found, node: check`; с `--configFile`
  прочие флаги и ENV молча игнорируются, «валидация» фактически запускает полноценный
  сервер и занимает сокеты entry points). Прежняя цепочка с `when: rc == 0` никогда не
  перезапускала сервис (rc=0 недостижим). Новая схема (план B): остановка сервиса →
  тайм-аутный пробный запуск `timeout -k ... traefik --configFile=...` (rc 124/137 +
  отсутствие строк ошибок = успех) → старт; при провале — откат на `<конфиг>.bak`,
  старт предыдущей конфигурации и прерывание плейбука с выводом пробы. Конфликт портов
  (`address already in use`) классифицируется как проблема окружения: без отката, с
  подсказкой. Проба и прочие side-effect-задачи пропускаются в Ansible check-mode.
- **Секреты вынесены из юнита**: `traefik_envs` рендерится в root-файл `traefik.env`
  (mode `0600`, задача с `no_log`) и подключается через `EnvironmentFile`; в world-readable
  юнит (0644) токены больше не попадают.
- Сравнение версий при установке бинарника стало точным (вместо подстроки): установлен
  `3.7.14`, запрошен `3.7.1` больше не считается «уже установленным». Извлечение версии
  выполняется без backref-аргумента `regex_search` (`'\\1'`): в части версий Ansible его
  разбор падал с `AttributeError: 'NoneType' object has no attribute 'group'`, а в других
  возвращал список, из-за чего сравнение всегда давало «не совпадает» и бинарник
  переустанавливался при каждом запуске. Используется строковой
  `regex_search('Version:\s*\S+')` + `regex_replace`; отсутствие совпадения → переустановка.
- Копирование бинарника теперь уведомляет `validate_and_restart_traefik` — новый демон
  реально подменяет работающий (перезапуск по-прежнему только после успешной пробы).
- Merge опций entry point-ов (`combine({'entryPoints': ...})`) стал рекурсивным: при
  добавлении одной опции (например prometheus-эндпоинта) без metrics-entrypoint раньше
  перетирались ВСЕ entryPoints из дефолтной статики.
- Убраны мёртвые `changed_when: *.stat.exists` и регистрирующие `stat` у задач
  `state: absent` (`file: state=absent` идемпотентен нативно); убран мёртвый `when`
  внутри элементов loop (условие per-item Ansible не поддерживает — каталог плагинов
  вынесен в отдельную задачу с `when: traefik_pilot_enabled`).
- `meta/main.yml`: исправлена лицензия `license(Apache-2.0)` → `Apache-2.0`.
- Molecule: версия в inventory приведена к `v3.7.14` (была `v3.6.2`), в static_config
  добавлены redirections web→websecure (verify проверяет 301-редирект, но override их
  не задавал), удалены сломанные шаги `side_effect` (playbook отсутствует) и
  `--check=true` в verify (заменён проверкой наличия откатного бэкапа — он создаётся
  только после успешной пробы), сравнение версии в verify сделано точным (также без
  backref-аргумента regex_search).
- Комментарии defaults/README/THINKING/AGENTS больше не утверждают, что пилот-токен
  можно передать через `traefik_envs` (ENV при указанном `--configFile` игнорируются)
  и что роль использует `--check`.
- Обновлена закреплённая версия Traefik `v3.6.2` → `v3.7.14` (defaults, README,
  molecule, CHANGELOG).
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
- **Откатная точка конфигурации**: переменная `traefik_config_backup_file`
  (`<конфиг>.bak`) — обновляется обработчиком только после успешной пробы и
  восстанавливается при провале; без неё сервис при битом конфиге не стартует.
- **Параметры пробы**: `traefik_validate_timeout` (сколько секунд пробный запуск
  должен пережить, по умолчанию 5) и `traefik_validate_kill_after` (отсрочка до
  SIGKILL, 10 с).
- **EnvironmentFile для секретов**: `traefik_env_file` (по умолчанию
  `/etc/traefik/traefik.env`, root, 0600) + задачи деплоя/очистки в `configure.yml`
  (`no_log`); юнит подключает его через `EnvironmentFile=` только при непустом
  `traefik_envs`.
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
- Переменная `traefik_version` с закреплением релиза (по умолчанию `v3.7.14`) и идемпотентная
  установка бинарного файла: архив с GitHub Releases загружается и распаковывается только при
  отсутствии бинарника или несовпадении установленной версии с запрошенной (точное
  сравнение через regex).
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
- Цепочка обработчиков `validate_and_restart_traefik`: перед любым перезапуском новая
  конфигурация проверяется тайм-аутным пробным запуском (см. «Исправлено» выше); при
  ошибке выполняется откат на `<конфиг>.bak`, плей прерывается, и сервис **никогда не
  перезапускается с некорректной конфигурацией**.
- Настройка logrotate (`/etc/logrotate.d/traefik`) с postrotate-сигналом `SIGUSR1` — Traefik
  переоткрывает лог-файлы без простоя и без перезапуска; параметры частоты, глубины ротации и
  максимального размера вынесены в переменные; ротацию можно отключить (`traefik_logrotate_enabled: false`).
- PID-файл сервиса в `/run/traefik/traefik.pid` (ExecStartPost + PIDFile) для точечной отправки SIGUSR1.
- Восстановление SELinux-контекста для бинарного файла Traefik при включённом SELinux.
- Molecule-сценарий (`molecule/default`): docker-драйвер, RockyLinux 9, последовательность
  dependency → destroy → syntax → create → prepare → converge → idempotence → verify → destroy.
- CI-воркфлоу публикации роли на Ansible Galaxy (`.github/workflows/release.yml`).

### Изменено
- Структура задач разбита на модули: `install.yml` → `configure.yml` → `systemd.yml` → `logrotate.yml`
  (входят в `tasks/main.yml` через `include_tasks`).
- Валидация конфигурации убрана из inline-параметра `validate:` шаблона: офлайн-валидатора
  у Traefik v3 нет, а пробный запуск занимает сокеты entry points, поэтому единственной
  авторитетной точкой проверки выбрана цепочка обработчиков перед перезапуском.
- Обновлены формулировки в README.md (переведены на русский, дополнены описанием переменных).

## [0.1.x] — исторические базовые версии
- Начальная версия роли: установка Traefik как systemd-сервиса с минимальным набором переменных
  (`traefik_version`, `traefik_service_name`, пути и учётная запись).
