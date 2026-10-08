# Примеры плейбуков: провайдеры Traefik
=====================================

В этом каталоге находятся готовые примеры плейбуков, показывающие как
подключать к роли `w0.traefik` (systemd-сервис Traefik v3) разные
**providers** — источники динамической конфигурации.

| Файл | Провайдер | Сценарий |
|------|-----------|----------|
| [`playbook-docker-provider.yml`](examples/playbook-docker-provider.yml) | `docker` | Одиночный хост с Docker Engine; сервисы объявляются LABEL'ами контейнеров |
| [`playbook-k3s-provider.yml`](examples/playbook-k3s-provider.yml) | `kubernetesprovider` | Сервер(ы) k3s; маршрутизация через Ingress / IngressClass и CRD Traefik |

Общие идеи
----------

1. **Merge вместо переопределения.** Роль хранит статическую конфигурацию в
   декларативном словаре `traefik_static_config`. Чтобы добавить провайдер и
   не потерять дефолты (entryPoints, логи, api, file-provider), используется
   рекурсивный `combine`. Базой merge служит **`traefik_role_defaults`** —
   plain-копия дефолтов роли (`vars/main.yml`):

   ```yaml
   traefik_static_config: >-
     {{ traefik_role_defaults.traefik_static_config
        | combine({'providers': {'docker': docker_provider_opts}},
                  recursive=True) }}
   ```

   Обратите внимание: опции провайдера добавляются **вложенно** под ключом
   `providers.docker:` (для Kubernetes — `providers.kubernetesprovider:`).
   Если писать `... .providers | combine(docker_provider_opts)`, опции
   (`endpoint`, `network` и т.п.) попадут в корень секции `providers:` вместо
   `providers.docker:`, и Traefik их проигнорирует.

   > ⚠️ Нельзя писать `{{ traefik_static_config | combine(...) }}` (или
   > `{{ traefik_static_config.providers | combine(...) }}`) внутри
   > собственного определения `traefik_static_config` (в `roles:` или `vars:`):
   > Ansible резолвит имя обратно в эту же переменную, возникает цикл
   > подстановки и задача «Generate static Traefik configuration from
   > traefik_static_config dict» падает с
   > `AnsibleError: ... maximum recursion depth exceeded while calling a
   > Python object`. Именно поэтому база для merge вынесена в отдельную
   > нешаблонную переменную `traefik_role_defaults` (а копия блока providers —
   > в `traefik_default_providers`, обе в `vars/main.yml`).
   >
   > Типичный «исправленный наполовину» вариант, который всё ещё ломается:
   >
   > ```yaml
   > # НЕПРАВИЛЬНО — базой снова выступает сама traefik_static_config:
   > traefik_static_config: >-
   >   {{ traefik_static_config
   >      | combine({'providers':
   >                   traefik_static_config.providers
   >                   | combine(docker_provider_opts)}, recursive=True) }}
   > ```
   >
   > Здесь рекурсия двойная: и корневой словарь, и `traefik_static_config.providers`
   > ссылаются на определяемую переменную. Заодно это и структурная ошибка:
   > `combine(docker_provider_opts)` вливает `endpoint/network/exposedByDefault`
   > прямо в корень `providers:` вместо `providers.docker:` — Traefik такие
   > ключи проигнорирует. Единственный корректный вариант — merge в
   > `traefik_role_defaults.traefik_static_config` с вложенным ключом
   > провайдера (см. код выше).

2. **File Provider остаётся полезным** даже при активном docker/kubernetes
   провайдере: в `traefik_dynamic_configs` держат глобальные TLS-опции и
   middlewares, на которые затем ссылаются из LABEL'ов (`...@file`) или
   Ingress-аннотаций.

3. **Безопасность юнита.** Закалённый systemd-юнит роли (`ProtectSystem`,
   `NoNewPrivileges`, ограниченные capabilities) требует точечных поблажек для
   доступа к `/var/run/docker.sock` или API kube-apiserver — в примерах они
   оформлены через `traefik_extra_capabilities` / `traefik_extra_unit_vars` и
   отдельный kubeconfig с сервисным токеном (не cluster-admin в проде —
   достаточно прав на чтение ingress/service/secret + обновление status).

4. **ACME.** В обоих примерах включён HTTP-01 через `traefik_enable_acme` и
   `traefik_cert_resolvers`; для DNS-01 токены прокидываются через
   `traefik_envs` (ansible-vault), а не в конфиг-файл.

5. **Плагины Traefik v3 (trial + Pilot).** Плагины — это Yaegi-скрипты, которые
   Traefik скачивает с pilot.traefik.io; офлайн-«установки» пакета не существует.
   Роль реализует установку целиком:
   * `traefik_pilot_enabled: true` + `traefik_pilot_token` (хранить в ansible-vault) —
     роль добавляет в статику блоки `pilot:` и `experimental.plugins:`;
   * `traefik_plugins: [{name, version, moduleName}]` — роль идемпотентно
     предзагружает исходники `traefik trial --download <name>@<version>` в кэш
     `traefik_plugins_dir` (переатрибуция каталога + `ReadWritePaths` в юните);
   * middleware на плагине объявляется в динамической конфигурации как
     `<name>@plugin` (docker-пример) или через CRD `Middleware`
     `traefik.io/v1alpha1` c аннотацией Ingress (k3s-пример).
   Предзагрузка нужна, чтобы сбой pilot.traefik.io не блокировал запуск демона
   (он подхватывает закэшированную копию); повторная загрузка происходит только
   при отсутствии кэша или смене версии в `traefik_plugins`.

Запуск
------

```bash
# роль должна быть доступна как w0.traefik (galaxy-имя) либо укажите путь
ansible-galaxy install w0.traefik          # или ANSIBLE_ROLES_PATH=../roles
# токен Pilot (для примеров с плагинами) — в vault:
ansible-vault edit group_vars/traefik/vault.yml      # traefik_pilot_token_vault: "TRIAL-..."
ansible-playbook -i inventory.ini --vault-id @prompt docs/examples/playbook-docker-provider.yml
ansible-playbook -i inventory.ini --vault-id @prompt docs/examples/playbook-k3s-provider.yml
```

Замечания по примерам
---------------------

* Docker-пример использует коллекцию `community.docker` (сеть `proxy` и
  демо-контейнер `traefik/whoami`). Добавьте её в `requirements.yml`, если
  используете эти таски: `ansible-galaxy collection install community.docker`.
* K3s-пример:
  * сначала удаляет встроенный Traefik из k3s (`k3s helm uninstall traefik -n kube-system`),
    иначе два ingress-контроллера будут конфликтовать (в k3s также можно
    поставить флаг установки `--disable=traefik`);
  * создаёт ServiceAccount `traefik` в `kube-system` и long-lived токен, из
    которого генерируется `/etc/traefik/k3s.kubeconfig` (права `0600`,
    владелец — пользователь сервиса);
  * `kubernetesprovider` ходит в API через `https://127.0.0.1:6443`.
* Хост-группы в примерах: `traefik` (docker) и `k3s_servers` (k3s) —
  приведите их к своему inventory.
* Проверка синтаксиса:

  ```bash
  ansible-playbook --syntax-check docs/examples/playbook-docker-provider.yml
  ansible-playbook --syntax-check docs/examples/playbook-k3s-provider.yml
  ```
