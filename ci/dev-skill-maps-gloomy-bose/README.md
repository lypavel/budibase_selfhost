
# DEV Skill Maps Gloomy Bose

Инструкция по деплою ПО в dev окружение `dev-skill-maps-gloomy-bose`.

## Первичный деплой в окружение

В облаке уже должны быть выделены ресурсы:
- Managed Postgres
- Managed Redis
- Private Bucket

Их описание: [dev-skill-maps-gloomy-bose](https://sirius-env-registry.website.yandexcloud.net/dev-skill-maps-gloomy-bose.html).

Для деплоя в кластер нужен установленный [Helm CLI](https://helm.sh/docs/intro/install/) и `kubectl`. Подключитесь к кластеру Kubernetes в namespace окружения.

### Настройка маршрутизации nginx

В окружении используется маршрутизация посредством nginx. При первичном деплое необходимо изменить конфигурацию маршрутизатора согласно настройкам проекта. Конфигурация nginx находится в ConfigMap `main-nginx-config`. Пример конфигурации:
```nginx
      user nginx;
      worker_processes  2;
      events {
        worker_connections  10240;
      }
      http {
        server {
          listen       80;
          server_name  localhost;
          client_max_body_size 50M;
          location / {
            proxy_pass http://proxy-service:10000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_buffer_size 16k;
            proxy_buffers 8 32k;
            proxy_busy_buffers_size 64k;
          }
        }
      }
```

### Ручной деплой

Добавьте репозиторий Budibase в `helm`:

```sh
helm repo add budibase https://budibase.github.io/budibase/
helm repo update
```

Укажите в файле `values.yaml` необходимую для установки информацию:

```yaml
ingress:
  enabled: false  # отключаем ingress, так как в кластере он уже настроен


couchdb:
  createAdminSecret: true
  adminUsername: "<couchdb_admin_username>"
  adminPassword: "<couchdb_admin_password>"
  cookieAuthSecret: "couchdb_cookie_secret"
  erlangCookie: "couchdb_erlang_cookie"
  persistentVolume:  # включаем персистентное хранилище для couchdb
    enabled: true
    size: '1Gi'  # объём хранилища

services:
  objectStore:
    minio: false  # отключаем minio, так как используется Object Storage
    accessKey: "<bucket_access_key>"
    secretKey: "<bucket_secret_key>"
    url: "<bucket_host>/<bucket_name>/"
  redis:
    enabled: false  # отключаем redis, так как он уже настроен извне
    port: <redis_port>
    url: "<redis_service_name>"
    password: "<redis_password>"
```

Установите Budibase в кластер:

```sh
helm install --namespace dev-skill-maps-gloomy-bose budibase budibase/budibase -f path/to/values.yaml
```

Перезапустите Deployment `nginx`:

```sh
kubectl rollout restart main-nginx -n dev-skill-maps-gloomy-bose
```

Теперь Budibase должен быть доступен по адресу: https://skill-maps.yc-sirius-dev.pelid.team/

При первом заходе на страницу вам будет предложено создать суперпользователя.

## Удаление инсталляции

Удалить релиз можно с помощью

```sh
helm uninstall --namespace dev-skill-maps-gloomy-bose budibase budibase/budibase
```

Удалить хранилище couchdb можно с помощью

```sh
kubectl delete pvc database-storage-budibase-couchdb-0 -n dev-skill-maps-gloomy-bose
```

**После удаления персистентного хранилища необходимо обязательно очистить кэш в Redis, иначе после установки budibase не предложит создать суперпользователя.**

Подключитесь к поду с Redis:

```sh
kubectl exec -n dev-skill-maps-gloomy-bose -it <redis-cache-pod-name> -- bash
```

Подключитесь к Redis:

```sh
redis-cli -a <redis_password>
```

Очистите кэш:

```sh
FLUSHDB
```

TODO:

Обновление образа Budibase(?)

Бэкапы couchdb(?)