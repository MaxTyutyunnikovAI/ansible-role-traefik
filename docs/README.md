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
   рекурсивный `combine`:

   ```yaml
   traefik_static_config: >-
     {{ traefik_static_config
        | combine({'providers':
                     traefik_static_config.providers
                     | combine(docker_provider_opts)}, recursive=True) }}
   ```

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

Запуск
------

```bash
# роль должна быть доступна как w0.traefik (galaxy-имя) либо укажите путь
ansible-galaxy install w0.traefik          # или ANSIBLE_ROLES_PATH=../roles
ansible-playbook -i inventory.ini docs/examples/playbook-docker-provider.yml
ansible-playbook -i inventory.ini docs/examples/playbook-k3s-provider.yml
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
