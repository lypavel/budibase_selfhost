Для деплоя в кластер нужен установленный [Helm CLI](https://helm.sh/docs/intro/install/).

Добавьте репозиторий Budibase в `helm`:

```sh
helm repo add budibase https://budibase.github.io/budibase/
helm repo update
```

Создайте файл `values.yaml` и укажите в нём необходимую для установки информацию:

```yaml
# отключаем ingress, так как в кластере он уже настроен
ingress:
  enabled: false


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

<!-- Создайте секреты для `couchdb`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: budibase-couchdb
  namespace: your_namespace
data:
  adminPassword: <base64_encrypted_couchdb_admin_password>
  adminUsername: <base64_encrypted_couchdb_admin_username>
  cookieAuthSecret: <base64_encrypted_cookie_auth_secret>
  erlangCookie: <base64_encrypted_erlang_cookie>
type: Opaque
``` -->

Установите Budibase в кластер:

```sh
helm install --namespace <your_namespace> budibase budibase/budibase -f path/to/values.yaml
```

В настройках `nginx` укажите `proxy_pass` на `proxy-service` и задайте максимальный размер тела POST запроса (необходимо для импорта приложений, т.к. стандартного значения в 1MB недостаточно).

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

Перезапустите Deployment `nginx`.

Теперь Budibase должен быть доступен по выделенному домену.

Удалить релиз можно с помощью

```sh
helm uninstall --namespace your_namespace budibase budibase/budibase
```

После удаления персистентного хранилища необходимо обязательно очистить кэш в Redis, иначе после установки budibase не предложит создать суперпользователя.

TODO:

Бэкапы couchdb(?)