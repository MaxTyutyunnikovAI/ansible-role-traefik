# THINKING.md — архитектурные решения и ход рассуждений

Документ фиксирует ключевые проектные решения роли `w0.traefik` (Traefik v3 + systemd),
обоснования, проверенные эмпирически нюансы и альтернативы, от которых отказались.

## 1. Декларативная конфигурация вместо императивных шаблонов
**Решение**: вся статическая конфигурация Traefik задаётся одним словарём
`traefik_static_config` и рендерится фильтром `to_nice_yaml(indent=2)`; динамическая —
списком `traefik_dynamic_configs` (`{name, content}` → файл `<name>.yml`).

**Почему**: пользователь переопределяет конфигурацию данными из inventory/group_vars,
а не копированием и правкой Jinja-шаблона. Поведение предсказуемо (merge/replace словарей),
роль остаётся универсальной без разрастания десятков отдельных переменных под каждый ключ Traefik.

**Альтернатива (отклонена)**: шаблон с множеством `{% if %}` по отдельным переменным —
плохо масштабируется и конфликтует с эволюцией схемы Traefik.

## 2. Валидация конфигурации: тайм-аутный пробный запуск вместо несуществующего `--check`
**Решение**: никаких инлайн-`validate:` у задачи template. Единственный авторитетный шлюз —
обработчик `validate_and_restart_traefik` (план B): остановить сервис → прогнать
`timeout -k <kill_after> <timeout> traefik --configFile=<файл>` → критерий успеха:
rc 124 (SIGTERM по тайм-ауту) или rc 137 (дошёл до SIGKILL) **и** ни одной строки
`"level":"error"` / `ERR[` в захваченном выводе. Успех → старт сервиса и обновление
откатного бэкапа `<конфиг>.bak`; провал → восстановление бэкапа (если он есть) и
старт предыдущей конфигурации, затем прерывание плейбука с выводом пробы. Без бэкапа
сервис не стартует (запуск заведомо битого конфига — лишь краш-цикл).

**Почему (проверено эмпирически на Traefik v3.7.14, бинарник с GitHub Releases)**:
- **`traefik --check` в v3 не существует**: без `--configFile` падает с
  `field not found, node: check` (rc=1); с `--configFile` флаг вообще игнорируется —
  FileLoader шорткьюбит разбор остальных аргументов и ENV. То есть «валидация» на самом
  деле **запускает полноценный сервер**, который занимает сокеты entry points и никогда
  не возвращает rc=0: валидный конфиг → rc=124 (висит до тайм-аута), занятые порты →
  rc=1 `address already in use`. Ровно поэтому и старая цепочка (validate → `when: rc == 0`
  → restart) никогда не перезапускала сервис: `rc == 0` недостижим.
- ENV-переменные (`TRAEFIK_LOG_LEVEL` и др.) при указанном `--configFile` тоже
  игнорируются (проверено: 0 DBG-строк при `TRAEFIK_LOG_LEVEL=DEBUG`) — обойти
  «запустить на проверку с другими настройками» нельзя.
- Проба выполняется **после остановки сервиса**: при работающем экземпляре порты заняты
  им самим → rc=1 «address already in use» ложно выглядел бы как ошибка конфигурации.
- Классификация окружения: rc=1 + `address already in use` — конфликт с **чужим**
  процессом (проблема хоста, не конфига): без отката, с явной подсказкой в сообщении
  об ошибке; сервис остаётся остановленным, пока порт не освободят.
- Побочные эффекты пробы (выполняется от root): создаёт лог-файлы и acme.json от root.
  Логи предварительно создаются от пользователя сервиса, acme.json пере-owner'ится после
  пробы. Поэтому все side-effect-задачи цепочки закрыты `when: not ansible_check_mode`.
- Порядок handler'ов: `Reload systemd daemon` объявлен первым — новые свойства юнита
  должны попасть в демон до stop/start; порядок handler'ов внутри одного уведомления =
  порядок объявления.
- Секреты пилот-токена: токен рендерится в сам конфиг (0640, владелец-сервис) — ENV его
  бы не доставил (см. выше), поэтому задача шаблона получает `no_log` при заданном токене.

**Отказались**: от inline `validate:` (бинарник не имеет валидатора), от probe до
остановки сервиса (ложные конфликты портов), от мгновенного `meta: flush_handlers`
после configure.yml (оставляем применение в конце плейбука — как и раньше).

**Нюанс по версии**: `traefik version` выводит версию **без** ведущей `v`
(`Version: 3.6.2`), поэтому сравнение при идемпотентной установке strip'ит `v` из тега
и извлекает версию точным regex (`Version:\s*(\S+)`) — подстрочное сравнение ловило
баг вида: установлен `3.7.14`, запрошен `3.7.1` → `3.7.1` является подстрокой →
установка пропускалась.

## 3. Ограничения офлайн-валидации динамической конфигурации
Динамические ресурсы File Provider **нельзя** безопасно валидировать офлайн: они
подхватываются рантаймом (`providers.file.watch: true`) без перезапуска, а TLS-опции
разрешаются только при реальном handshake. Поэтому валидируется только статика — это
осознанное, документированное ограничение, а не упущение.

## 4. Идемпотентная установка бинарника
Загрузка и распаковка tarball выполняются только если бинарник отсутствует ИЛИ его версия
не совпадает с `traefik_version` (проверка через `stat` + `traefik version` c точным
regex-извлечением версии, обе задачи с `changed_when: false` и `check_mode: false`).
Апгрейд бинарника уведомляет handler-chain: новый бинарник должен реально заменить
работающий демон, но перезапуск всё равно идёт только после успешной пробы конфигурации.
Временный каталог загрузки очищается после установки.

## 5. Закалка systemd-юнита
Базовая триада обязательна: `NoNewPrivileges`, `ProtectSystem=full`, `ProtectHome`,
`PrivateTmp`. Дополнительно: `PrivateDevices`, `ProtectKernelTunables/Modules`,
`RestrictSUIDSGID/Namespaces`, `LockPersonality`, `SystemCallArchitectures=native`,
`StartLimitBurst=3` (краш-цикл не должен «ронять» машину).

Привилегированные порты :80/:443 от непривилегированного пользователя `traefik` — через
`CAP_NET_BIND_SERVICE` (CapabilityBoundingSet + AmbientCapabilities), а не запуск от root.
`ReadWritePaths` whitelists только логи и data-каталог. `MemoryDenyWriteExecute=false`
намеренно: Go-runtime может использовать JIT-подобные аллокации, strict-режим ломает сервис.

## 6. Ротация логов через SIGUSR1 вместо copytruncate
Traefik нативно переоткрывает лог-файлы по `SIGUSR1` — postrotate в logrotate шлёт сигнал
процессу (pid-файл `/run/traefik/traefik.pid`, созданный `ExecStartPost`; pgrep как fallback).
`copytruncate` сознательно **не** используется: оно может терять строки между truncate и
копированием. Смена конфигурации logrotate никогда не требует рестарта самого Traefik.

## 7. Модульная структура задач
`tasks/main.yml` подключает `install → configure → systemd → logrotate` через `include_tasks`.
Модули читаются и тестируются независимо; условие `when: traefik_logrotate_enabled` вынесено
на уровень подключения модуля. Динамические `include_tasks` (а не `import`) выбраны для
независимости и гибкости.

## 8. Прочие решения
- **Отдельный системный пользователь** с `nologin` и `create_home: false`; home указывает
  на `/var/lib/traefik` (acme.json и runtime-состояние).
- **Режимы 0640/0750** для конфигов и каталогов — минимум привилегий; бинарник root:root 0755.
- **SELinux**: `restorecon` для бинарника только когда SELinux включён (и не в check-mode).
- **Архитектуры**: маппинг `ansible_facts.architecture` → имена Go/Release (`amd64/arm64/386`)
  в `vars/main.yml` с graceful fallback.
- **Molecule на RockyLinux 9**: systemd в контейнере требует privileged + tmpfs /run +
  монтирование cgroup; минимальная подготовка хоста (tar/gzip/procps-ng/logrotate) — в pre_tasks.

## 9. Плагины Traefik v3: pilot + предзагрузка trial-кэша
**Решение**: установка плагина в роли — это три связанных шага: (1) статический блок
`pilot:` (token + доп. свойства из `traefik_pilot_extra_props`) и `experimental.plugins:`
(`<name>: {moduleName, version}`), которые шаблон `traefik.yml.j2` генерирует из
`traefik_pilot_enabled` / `traefik_plugins`; (2) идемпотентная предзагрузка исходников
`traefik trial --download <name>@<version>` (`tasks/configure.yml`, `creates:` на файл
кэша `<name>@<version>.go`); (3) middleware, ссылающийся на плагин (`<name>@plugin` в file
provider или CRD Middleware в k8s).

**Почему**: у Traefik v3 нет офлайн-установки плагинов — демона можно «накормить» только
через pilot.traefik.io. Предзагрузка в `traefik_plugins_dir` гарантирует, что сбой/отсутствие
доступа к pilot не блокирует запуск (daemon использует закэшированную копию), а `creates:`
делает задачу идемпотентной: повторная загрузка — только при чистом кэше или смене версии.
Токен попадает в траекторию командной строки задачи, поэтому `no_log: true`; сам конфиг-файл
имеет режим 0640 и принадлежит сервисному пользователю. При включённом pilot юнит получает
`ReadWritePaths={{ traefik_plugins_dir }}` — иначе `ProtectSystem` запретит daemon'у писать кэш.

**Альтернативы (отклонены)**: отдельная роль/плейбук для плагинов (дублирование логики
статической конфигурации); скачивание через `get_url` с GitHub (не соответствует контрактной
механике trial и формату кэша daemon'а).

## 10. Kubernetes-провайдеры: подключение без kubeconfig и строгий парсинг
**Решение**: k8s-провайдеры — первоклассенные переменные `traefik_kubernetes_*`,
которые шаблон `traefik.yml.j2` вливает в существующий блок `providers` через
`combine(..., recursive=True)` (file/docker и прочее не теряются). Connection =
`endpoint` + `token` + `certAuthFilePath`; per-provider опции вынесены в отдельные
словари `ingress/crd/gateway_extra`, потому что Traefik парсит статику строго.

**Эмпирика (бинарник v3.7.14)**:
- поля `kubeconfig` в статике НЕ существует: `{"level":"error","error":"field not
  found, node: kubeconfig"}`, rc=1 → вся механика записи kubeconfig-файля ролью
  отклонена; рабочие альтернативы — endpoint+token+certAuthFilePath (используется)
  или env `KUBECONFIG` через `traefik_envs`;
- валидность подтверждена прогоном: `ingressClass` у Ingress/CRD,
  `experimentalChannel` у Gateway, `namespaces`, per-provider extras — конфиг
  принимается (rc=124 по тайм-ауту, ноль строк ошибок);
- ошибки инициализации провайдеров (отсутствующий CA, недоступный API) пишутся
  ТОЛЬКО в `log.filePath` и не видны в stdout/stderr, демон продолжает работать
  → проба перезапуска роли не видит состояние кластера и не делает ложный откат.

**Почему per-provider extras**: `allowExternalNameServices`/`allowEmptyServices`
валидны только для `kubernetesCRD`; общий `extra`-словарь для всех трёх провайдеров
рано или поздно приведёт к `field not found` и откату конфигурации.

**Альтернативы (отклонены)**: обвязка kubeconfig-файлами (поле отсутствует в v3);
плоские переменные вида `traefik_kubernetes_crd_allow_external_name_services`
(дублирование ×3 и жёсткая привязка к конкретным опциям, перенос в `*_extra` не
требует изменения шаблона).

**Кластерные манифесты — вне роли**: объект IngressClass, Gateway API CRDs и
RBAC настраиваются плейбуком/инвентарём (пример `docs/examples/
playbook-k3s-provider.yml`); роль только рендерит конфиг.

## 11. Дашборд: алиас `traefik_dashboard_users` вместо переименования
> **Устарело (см. §21)**: алиасы `traefik_dashboard_users` /
> `traefik_dashboard_basic_auth` удалены при упорядочивании basicauth.
> Запись сохранена как история решения.
>
- Роль traefik2/rolehippie питает basicAuth дашборда переменной
  `traefik_dashboard_users` (список htpasswd-строк `user:hash`); в traefik3 та же
  семантика называется `traefik_dashboard_basic_auth`. Два имени на одно свойство —
  источник путаницы при миграции плейбуков.
- Решение: **алиас, а не переименование** — правило AGENTS «не ломать контракт
  переменных»: `traefik_dashboard_basic_auth` остаётся каноническим именем, его
  default становится мостом `"{{ traefik_dashboard_users }}"`. Задан только один
  из двух — работает любой; заданы оба — выигрывает `traefik_dashboard_basic_auth`
  (обычная precedence варсов, без combine-магии).
- `realm: Traefik` добавлен в middleware — окно авторизации подписывается так же,
  как в traefik2 (косметика, но единообразие миграции).
- ipAllowList (`traefik_dashboard_trusted_ips`) в первом подключении сознательно
  не задавался: middleware `internal` рендерится только при непустом списке, а
  ссылка на непустой `traefik_dashboard_middlewares` с несуществующим middleware
  ломает роутер — список и источники должны меняться парно (это задокументировано
  в комментарии плейбука `playbooks/server-ru/_traefik.yml`).

## 12. RBAC — в роли, а не в отдельной обёртке
**Решение**: автоматическое создание ServiceAccount, ClusterRole, токена и CA
реализовано опциональным модулем `tasks/kubernetes.yml` внутри той же роли, с
master switch `traefik_kubernetes_rbac_enabled: false`.

**Почему**: (1) RBAC — это подготовка данных для переменных роли (`traefik_kubernetes_token`,
`traefik_kubernetes_cert_auth_file`); вынесение в отдельную роль создало бы циклическую
или жёстко-упорядоченную зависимость с дублированием имён переменных. (2) Ресурсы
кластера (SA, ClusterRole, Secret) — не внутрикаталожные артефакты Traefik, а внешние
объекты K8s, но их создание — первый шаг в сценарии «Traefik снаружи K3s», и удобно
иметь его под одним entry point. (3) `set_fact` напрямую передаёт значения в
`traefik_kubernetes_token`/`cert_auth_file`, что гарантирует согласованность.

**Альтернативы (отклонены)**: отдельная wrapper-роль (два релиза, дублирование doc;
переменные роли Traefik нужно либо дублировать, либо использовать `include_role` с vars);
ручное создание через `k3s kubectl` в плейбуке (пример playbook-k3s-provider.yml — работало,
но каждый пользователь переписывал свои таски с kubectl).

## 13. Доставка токена: путь к файлу, не значение
**Решение**: токен записывается в файл `{{ traefik_k8s_token_file }}` (права `0640`
traefik:traefik), а set_fact передаёт **путь** к этому файлу. Traefik v3.7+ умеет
читать `providers.kubernetes*.token` как путь.

**Почему**: (1) токен никогда не появляется в статической конфигурации
`traefik.yml` (шаблон получает `traefik_kubernetes_token = "/etc/traefik/k8s-token"`,
а не сам JWT). (2) при ротации токена достаточно обновить файл и перезапустить
сервис — конфиг не меняется (не нужен probe-цикл обновления статики). (3) выписанный
в файл токен можно опционально скопировать на контроллер (`delegate_to: localhost`)
для шифрования `ansible-vault encrypt` без вывода в логи.

**Альтернативы (отклонены)**: передача токена set_fact (значение попадает в шаблон →
в rendered traefik.yml, задача идёт с no_log, но токен оседает на диске в конфиге —
избыточная экспозиция; отказ vault workflow). Автоматическая запись в ansible-vault
на контроллере (Ansible не умеет шифровать vault-файлы изнутри playbook без внешних
инструментов).

## 14. CA из kubeconfig, не из client-ca.crt
**Решение**: CA-сертификат извлекается из kubeconfig-файла (поле
`certificate-authority-data` — base64-кодированный PEM — или поле
`certificate-authority` — путь к файлу). Fallback — ConfigMap `kube-root-ca.crt`.

**Почему**: kubeconfig содержит именно тот CA, который служит для проверки
сертификата kube-apiserver’а. В k3s файл `/var/lib/rancher/k3s/server/tls/client-ca.crt`
(использованный в старом примере) — это CA для клиентских сертификатов, не
обязательно подписывающий serving-сертификат API. Использование CA из
kubeconfig гарантирует, что Traefik сможет верифицировать TLS endpoint.

**Альтернатива (отклонена)**: копирование `/var/lib/rancher/k3s/server/tls/server-ca.crt`
(зависит от версии k3s, не гарантирован на всех хостах). ConfigMap
`kube-root-ca.crt` (присутствует с k8s ≥ 1.21; но не содержит CA, если кэш
корневого CA отключён/настроен нестандартно).

## 15. TLSStore CRD вместо file provider для дефолтного сертификата
**Решение**: создание ресурса `TLSStore` (`traefik.io/v1alpha1`) со ссылкой на
Kubernetes Secret (`spec.defaultCertificate.secretName`). Дополнительно —
file-based fallback через `traefik_default_certificate`.

**Почему**: File Provider читает TLS-сертификаты из файловой системы
(`certFile`/`keyFile`), но cert-manager хранит их в Kubernetes Secrets. Единственный
нативный способ для Traefik получить сертификат из Secret — CRD TLSStore
(kubernetesCRD provider). SecretName указывает на Secret, где cert-manager
сохраняет ключи (`tls.crt`/`tls.key`/`ca.crt`).

**Ограничение**: TLSStore требует установленных CRD `traefik.io` и активного
провайдера kubernetesCRD. Роль проверяет CRD перед созданием (`k8s_info` с retry)
и даёт понятный hint в случае отсутствия.

## 16. Htpasswd: apr_md5_crypt вместо bcrypt
**Решение**: модуль `community.general.htpasswd` со схемой `apr_md5_crypt` (MD5-crypt,
префикс `$apr1$`). Требует `python3-passlib` (чистый Python, без bcrypt-зависимостей).

**Почему**: Traefik официально поддерживает MD5 `$apr1$` (документация Traefik
показывает именно `$apr1$` в примерах basicAuth). `apr_md5_crypt` — это схема из
passlib, не требующая дополнительных библиотек (pure Python). bcrypt потребовал бы
`python3-bcrypt` + компиляцию на целевом хосте, что тяжелее для дефолтного сценария.

## 17. Пресеты middleware: цепочка и независимые переключатели
**Решение**: три независимые переменные `traefik_enable_*` (security-headers,
rate-limit, dashboard-auth), каждая добавляет middleware-определение в
`00-default-middlewares.yml` и, если включено, включает middleware в цепочку
`web-chain` (`chain.middlewares`).

**Почему**: (1) независимые переключатели позволяют включать только нужное —
security-headers по умолчанию, rate-limit опционально. (2) `web-chain` — это
отдельное middleware типа `chain`, собирающее активные пресеты; роутеры
подключают одну точку: `middlewares: ["web-chain@file"]`. (3) dashboard-auth
намеренно НЕ включается в общую цепь — он должен применяться только к
дашборду/API. (4) `traefik_default_middlewares_enabled` оставлен для обратной
совместимости: его словарь рендерится наряду с новыми пресетами.

**Альтернативы (отклонены)**: единый словарь `traefik_middlewares_presets`
(меньше переменных, но сложнее комбинировать с легаси-настройками).
Жёстко зашитый chain (каждый новый пресет молниеносно применяется ко всем
роутерам — конфликт с безопасностью).

## 18. Диагностика 422 при создании ClusterRoleBinding: отрезанный `spec`
**Решение**: в задачах `Create ClusterRole` и `Bind the ClusterRole to the ServiceAccount`
убран уровень `spec:` — правила (`rules`), `roleRef` и `subjects` размещены на верхнем
уровне определения, как требует схема RBAC `rbac.authorization.k8s.io/v1`.

**Симптомы**: (1) фатал 422 при создании ClusterRoleBinding —
`roleRef.kind: Unsupported value: ""`, `roleRef.name: Required value`;
(2) в логе прогона на задаче ClusterRole — `[WARNING]: unknown field "spec"`;
(3) после прогона `kubectl get clusterrole traefik-external -o yaml` → `rules: null`
(роль создана пустой, SA фактически без прав).

**Почему так**: у ClusterRole/ClusterRoleBinding/Role/RoleBinding (и у всего RBAC `v1`)
нет поля `spec` — в отличие от обычных ресурсов. API-сервер Kubernetes принимает запрос
с `fieldValidation=Warn`, выдаёт warning `unknown field "spec"` и **отрезает** это поле
(prune). Для ClusterRole это тихо удаляло `rules` (задача уходила в `changed`); для
ClusterRoleBinding вместе с `spec` исчезал `roleRef` целиком, и валидация падала на
пустом `kind`/`name`. Диагностика подтвердила, что порча НЕ в ansible-транспорте/клиенте:
warning доказывает, что payload дошёл до сервера полным, а статический аудит (YAML-парс,
код коллекции `kubernetes.core`, сериализация клиента kubernetes 37.0.1) чист.

**Альтернативы (отклонены)**: передавать `resource_definition` строкой YAML (мимо
словарного транспорта) — избыточно, т.к. механизм установлен; пересоздавать объекты
руками — не декларативно.

## 19. Токен Kubernetes: b64decode и запрет завершающего LF
**Решение**: факт токена вычисляется как
`{{ _token_secret.resources[0].data.token | b64decode | trim }}`; файлы
(рабочий `k8s-token` и дамп для vault) записываются **без** завершающего `\n`.

**Симптомы** (после починки RBAC, см. §18): в логе Traefik бесконечные
`reflector.go: Failed to watch ... net/http: invalid header field value for
"Authorization"` — провайдеры kubernetes не работают.

**Диагностика (улики)**:
1. `head /etc/traefik/k8s-token` → `ZXlKaGJHY2l...` — это base64 самого JWT:
   `kubernetes.core.k8s_info` отдаёт `data` Secret без декодирования (как в API),
   роль же писала значение как есть → Traefik слал бы base64 вместо подписи (401).
2. `od -An -tx1` последних байтов файла → `... 64 77 3d 3d 0a` = `dw==\n` —
   завершающий LF из `content: "{{ ... }}\n"` (YAML двойные кавычки → реальный LF).
3. `curl -H "Authorization: Bearer $(tr -d '\n' < k8s-token)" .../api/v1/namespaces`
   → 401: подтверждает, что одного обрезания LF недостаточно — нужен и decode.
4. Сама ошибка «invalid header» — это про LF (client-go отвергает управляющие
   символы в значении заголовка), она срабатывает раньше любой проверки кредов.

**Почему две ошибки маскировали друг друга**: LF давал симптом сразу (валидация
заголовка), base64 — только после устранения LF (401). Факт с `b64decode | trim`
— единственная точка исправления, broadcast/запись/дамп потребляют его уже
очищенным. Убранное `\n` безопасно: JWT однострочный, а дамп в vault используется
через `$(cat file)`, где LF тоже сломал бы сценарий.

## 20. Права ClusterRole: минимальный набор = фактический набор watch
**Решение**: в `_traefik_rbac_base` добавлены пять ресурсов, дававших 403:
`configmaps` и `nodes` (core), `endpointslices` (`discovery.k8s.io`),
`middlewaretcps` и `serverstransporttcps` (`traefik.io`).

**Почему именно они**: после починки токена (§19) креды стали валидными, и
`journalctl -u traefik` выдал полный набор отказов — 41× `cannot list resource`,
после дедупликации 4 уникальных ресурса; пятый (`nodes`) проявился только
после того, как первые четыре были выданы и Traefik перезапущен (informer
узлов, видимо, стартует после успешных остальных). Их отсутствие в правилах —
рассинхрон между тем, что смотрит Traefik v3.7, и тем, что перечислено в роли:

- Traefik v3.7 (client-go 0.36) использует EndpointSlice вместо простых Endpoints
  для построения сервисов — права были только на `endpoints`;
- `configmaps` — informer провайдеров (label-selector `owner!=helm`) смотрит
  cluster-wide, а не только в ns Traefik;
- `middlewaretcps`/`serverstransporttcps` — TCP-варианты CRD, которые роль
  сама не создавала, но они есть в CRD-манифесте `traefik-crd-v3.yml`
  (10 kind'ов traefik.io) — список прав отставал от манифеста;
- `nodes` — подтверждено официальным `kubernetes-crd-rbac.yml` Traefik v3.7.14
  (там nodes входит в базовый core-блок наравне с services/secrets/configmaps).

**Принцип**: источник истины — и фактические запросы informer'ов в логах, и
официальный upstream-RBAC (сверены, совпали); `pods` из upstream сознательно
не добавлен (нужен только для OTel-атрибутов), legacy-группа
`traefik.containo.us` и gateway-права оставлены без изменений (ошибок нет).

## 21. Упорядочивание basic-auth: один режим usersFile, четыре переменные
**Решение**: откат §11 и упразднение трёх режимов авторизации. Остаются
`traefik_dashboard_auth_enabled` (бывш. `traefik_enable_dashboard_basic_auth`),
`traefik_dashboard_user` (бывш. `..._admin_user`), `traefik_dashboard_password`
(бывш. `..._admin_password`) и internal-путь `traefik_dashboard_auth_file`
(бывш. `..._htpasswd_file`). Удалены: алиасы `traefik_dashboard_users` /
`traefik_dashboard_basic_auth` (inline-режим), автогенерация пароля и файл
`dashboard-admin-password`. Один assert (пароль обязателен при auth_enabled),
одна задача `htpasswd`, рендер всегда через `usersFile`.

**Почему**:
1. Три режима (inline / vault / автогенерация) с приоритетами — источник
   инцидентов: плейбук держал старый inline bcrypt-хеш, пресет с
   `traefik_dashboard_admin_password` молча игнорировался, и ни один пароль
   не работал (диагностика: `curl -u admin:admin333` → 401 при живом realm).
2. Inline-хеши лежали в `00-default-middlewares.yml` — секреты в динамической
   конфигурации; usersFile (0640) — правильное место.
3. Автогенерация (рандом + `dashboard-admin-password`) добавляла две задачи,
   slurp/stat-логику и ещё один файл-состояние; по требованию владельца
   удалена — пароль всегда явный (vault).
4. Middleware `dashboard-auth` теперь рендерится только вместе с
   `traefik_dashboard_enabled`: роутер без авторизации невозможен при
   auth_enabled, а bare-установка (дашборд выключен) не требует пароля —
   assert не срабатывает без причины.

**Альтернативы (отклонены)**: deprecated-warn на релиз (дольше жить с двумя
источниками правды); оставить inline как «продвинутый» режим (возвращает
приоритеты и инцидент-класс №1); оставить автогенерацию (меньше кода, но
пароль вне vault и лишний файл-состояние).

## 22. Плагины без pilot и `traefik_dynamic_configs_extra`
**Решение**: (1) шаблон статики рендерит `experimental.plugins` по одному
условию `traefik_plugins | length > 0` (раньше — `traefik_pilot_enabled and
traefik_plugins`), merge — `recursive=True`; (2) новый список
`traefik_dynamic_configs_extra` дополняет `traefik_dynamic_configs` в том же
цикле `configure.yml`.

**Почему**:
1. Pilot/trial — механизм Traefik v2: блок `pilot` и команда `traefik trial`
   в v3 удалены, а фактическая установка плагина на v3 — одна декларация
   `experimental.plugins` (daemon сам скачивает модуль из каталога на старте).
   Прежнее двойное условие делало `traefik_plugins` мёртвой переменной на v3
   при дефолтном `traefik_pilot_enabled: false` — плейбук не мог включить
   плагин, не включая несуществующий в v3 пилот (и задачу `trial`, которая
   на v3 упала бы). Guard задачи предзагрузки и `pilot:`-блок оставлены как
   v2-обратная совместимость.
2. `traefik_dynamic_configs_extra` нужен, потому что плейбуковый override
   `traefik_dynamic_configs` ЗАМЕНЯЕТ весь default-список (priority
   defaults < play) — плейбук с одним своим item молча выпилил бы встроенный
   `tls-options` (minVersion/sniStrict/cipherSuites) из дальнейшего рендера.
   Append-вариант сохраняет и default, и плейбуковый контент.
3. Контекст применения — перенос `my-robots-txt` (плагин
   `github.com/solution-libre/traefik-plugin-robots-txt`) с server-ni на
   server-ru: ссылки `my-robots-txt@file` в IngressRoute при отсутствии
   определения заставляли Traefik молча отбрасывать роутер целиком
   (роутер/сервис не появляются в API, ACME не запускается, sniStrict рвёт
   handshake — «неработающие» whoami и k3s-dashboard).

**Альтернативы (отклонены)**: инъекция `experimental.plugins` вручную в
`traefik_static_config` плейбука (обходит предназначение переменной
`traefik_plugins`, сервер-ni-стиль, но плейбуковый merge пришлось бы держать
синхронно с шаблоном); встроить item `my-robots-txt` в defaults роли
(проектный контент в role-слой — он по определению env-специфичный);
`traefik_dynamic_configs_template` под файл роботов (один шаблон на роль —
тривиально конфликтует с будущими генерациями); пилот(true)+trial (на v3.7
команды нет — задача упала бы).

## 23. Проба отклонила `plugins`: пустой узел и `{% set %}` в цикле

**Симптом** (прогон `playbooks/server-ru/_traefik.yml`, handler-проба):

```
command traefik error: plugins cannot be a standalone element
(type map[string]plugins.Descriptor)
```

Роль откатила конфиг на `.bak` и упала — сервис остался на прежней
конфигурации.

**Диагноз** (две независимые проверки):

1. *Источник сообщения* — `traefik/paerser`, `parser/nodes_metadata.go:81`:
   ```go
   if len(node.Children) == 0 && tagValue != TagLabelAllowEmpty && tagValue != "-" {
       return fmt.Errorf("%s cannot be a standalone element (type %s)", node.Name, fType)
   }
   ```
   «standalone» здесь = **пустой узел**: у `plugins` нет детей, а тип
   `map[string]plugins.Descriptor` не помечен `allowEmpty`. То есть Traefik
   отклонил не вложенность, а **пустизну** `plugins: {}`.
2. *Рендер шаблона в памяти* (defaults + vars + role-args плейбука, чистый
   Jinja2 с `combine`/`to_nice_yaml`): `experimental = {'plugins': {}}` —
   блок есть, накопление пустое.

**Корень**: в `traefik.yml.j2` accumulator собирался так:

```jinja
{% set _plugins_dict = {} -%}
{% for _p in traefik_plugins -%}
{% set _plugins_dict = _plugins_dict | combine({ _p.name: {...} }) -%}
{% endfor -%}
```

В Jinja2 `{% set %}` внутри `{% for %}` создаёт переменную в кадре итерации
и **сбрасывается в конце каждой итерации** — после цикла внешний
`_plugins_dict` остаётся `{}` (мини-репро: `{% set d = {} %}{% for k in ['a'] %}{% set d = d|combine({k:1}) %}{% endfor %}{{ d }}` → `{}`).
Баг латентный: раньше условие `traefik_plugins | length > 0` почти всегда
было ложным (переменная не задавалась), и мёртвый код не выполнялся.
Причина, по которой ошибка всплыла именно сейчас: `_traefik.yml` впервые
передал непустой `traefik_plugins`.

**Фикс**: накопление через `namespace(d={})` (канонический механизм мутации
за пределами цикла) + условие рендера заменено на
`_plugins_dict | length > 0` — пустой блок `experimental` не появляется ни
при каком стечении обстоятельств (защита и от paerser, и от лишнего ключа).

**Проверка**: in-memory рендер →
`experimental.plugins.robots-txt = {moduleName: github.com/solution-libre/traefik-plugin-robots-txt, version: v0.2.2}`,
один YAML-документ, корневого `plugins` нет; при `traefik_plugins: []`
блок `experimental` отсутствует; `to_nice_yaml` парсится обратно.

**Контингент при следующем прогоне**: проба выполняет реальный старт Traefik,
теперь с загрузкой модуля из каталога (proxy.golang.org / github). Если
загрузчик не доберётся до сети, в выводе пробы появится строка уровня
`error` — handler сочтёт конфигурацию сломанной и откатит (правильно:
без плагина middleware `my-robots-txt` невалиден). Действие при этом —
отдельная диагностика сети/кэша, а не откат самого фикса.
