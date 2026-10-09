# TODO — список задач

Роль: `w0.traefik` (Ansible, Traefik v3 как systemd-сервис).

## Высокий приоритет
- [ ] Добавить интеграцию Let's Encrypt (ACME): сертификат-резолвер `letsencrypt` указан в конфигурации по умолчанию, но `certificatesResolvers` в `traefik_static_config` не настроен — без него `websecure` не сможет получить сертификаты.
- [ ] Расширить тесты: verify-сценарии Molecule (идемпотентность после смены версии, поведение при битом конфиге).

## Средний приоритет
- [ ] Опциональная настройка middleware по умолчанию (compress, headers, rate-limit) в динамической конфигурации.
- [ ] Документировать схему миграции с предыдущих версий роли (изменения структуры переменных) в README.

## Низкий приоритет
- [ ] Метрики (Prometheus entryPoint) как опция через переменную.
- [ ] Вариант установки через пакетный менеджер (apt/yum) вместо загрузки бинарника с GitHub.
- [ ] Поддержка нескольких архитектур контейнеров при тестировании (arm64 в CI).
- [ ] Автоматическая проверка выхода новых релизов Traefik (renovate/dependabot для `traefik_version`).

## Закрыто / выполнено
- [x] Модульная структура задач: install → configure → systemd → logrotate.
- [x] Идемпотентная установка бинарного файла с проверкой версии (теперь — точное
  regex-сравнение версии, подстрочное сравнение ловило баг вида 3.7.1 ⊂ 3.7.14).
- [x] Декларативная статическая и динамическая конфигурация (File Provider).
- [x] Закалённый systemd-юнит с `CAP_NET_BIND_SERVICE`.
- [x] Перезапуск только после успешной валидации конфигурации — цепочка обработчиков
  переработана под Traefik v3: `--check` там не существует, валидация стала тайм-аутным
  пробным запуском с откатом на `<конфиг>.bak` (см. `THINKING.md`, раздел 2).
- [x] Секреты `traefik_envs` — в root-файле `traefik.env` (0600) через `EnvironmentFile`
  вместо `Environment=` в world-readable юните (0644).
- [x] Закреплённая версия Traefik обновлена до `v3.7.14` (включая molecule/README/CHANGELOG).
- [x] Применение `traefik_extra_unit_vars` в шаблоне `traefik.service.j2`.
- [x] Поддержка Docker/Kubernetes provider'ов — через документированные примеры merge в `traefik_static_config` (`docs/examples/`).
- [x] Установка плагинов Traefik v3 (pilot + experimental.plugins + предзагрузка `traefik trial --download`).
- [x] Logrotate с SIGUSR1 без перезапуска сервиса.
- [x] Molecule-сценарий на RockyLinux 9.
- [x] Перевод документации (README/CHANGELOG/TODO/AGENTS/THINKING) на русский язык.
