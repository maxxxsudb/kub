# Budibase для тестового Kubernetes

Зонтичный чарт включает исходники официального чарта Budibase из [v3.48.0](https://github.com/Budibase/budibase/tree/v3.48.0/charts/budibase) и его зависимость CouchDB 4.5.6. Чарт запускает несколько Pod: приложения, worker, proxy, CouchDB, Redis и MinIO. Дополнительно CouchDB использует init-контейнер. Официальный шаблон MinIO изменён: добавлена настройка `services.objectStore.image` и `IfNotPresent` для переноса образа в локальный registry. Исходный `Chart.yaml` имеет версию `0.0.0`, поскольку версия проставляется при официальной упаковке; версия приложения задана в `values.yaml`.

## Подготовка без интернета

Скопируйте всю папку `budibase` в закрытую сеть. Для установки не нужны `helm repo add` и `helm dependency update`. Образы в Git не входят; заранее загрузите их под архитектуру узлов и перенесите во внутренний registry:

```text
docker.io/budibase/apps:v3.48.0
docker.io/budibase/worker:v3.48.0
docker.io/budibase/proxy:v3.48.0
docker.io/budibase/database:2.1.0
docker.io/library/redis:7.4.5
docker.io/library/busybox:1.36.1
quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z
```

Например, `docker pull`, `docker save`, перенос файла, `docker load`, затем `docker tag` и `docker push` во внутренний registry. В `local-values.yaml` укажите фактические имена перенесённых образов:

```yaml
budibase:
  services:
    apps:
      image: registry.example.local/budibase/apps:v3.48.0
    automationWorkers:
      image: registry.example.local/budibase/apps:v3.48.0
    worker:
      image: registry.example.local/budibase/worker:v3.48.0
    proxy:
      image: registry.example.local/budibase/proxy:v3.48.0
    redis:
      image: registry.example.local/library/redis:7.4.5
    objectStore:
      image: registry.example.local/minio/minio:RELEASE.2025-09-07T16-13-09Z
  couchdb:
    image:
      repository: registry.example.local/budibase/database
      tag: "2.1.0"
    initImage:
      repository: registry.example.local/library/busybox
      tag: "1.36.1"
```

Если registry требует авторизацию, создайте в namespace `imagePullSecret` и укажите его в `budibase.imagePullSecrets`. Для доступа к registry должны быть настроены DNS, сертификаты и сетевой доступ узлов.

## Установка

Нужен Helm 3 и StorageClass по умолчанию с динамическим выделением томов. По умолчанию создаются PVC для CouchDB (10Gi), MinIO (2Gi) и Redis (100Mi). При необходимости задайте StorageClass и размеры через `budibase.couchdb.persistentVolume`, `budibase.services.objectStore` и `budibase.services.redis` в `local-values.yaml`. Для первого запуска официальный пример ориентируется примерно на 2 CPU и 6 ГБ памяти; фактические потребности зависят от нагрузки.

```sh
helm upgrade --install budibase ./budibase -n budibase --create-namespace -f local-values.yaml
kubectl get pods,pvc -n budibase
kubectl port-forward -n budibase svc/proxy-service 10000:10000
```

Откройте http://localhost:10000 и создайте первую учётную запись. Для доступа коллег настройте Ingress или тип Service под свой кластер; по умолчанию Ingress выключен. Полный список параметров — в `charts/budibase/values.yaml`. Шаблоны используют фиксированные имена сервисов: устанавливайте один экземпляр чарта в namespace.

При первом запуске чарт создаёт Kubernetes Secrets. Сохраняйте их вместе с PVC при переносе или восстановлении: в них находятся ключи шифрования и учётные данные. `helm uninstall` удаляет создаваемые чартом ресурсы, включая Secrets; перед удалением сделайте резервную копию данных и ключей. Секреты не задавайте в Git. Это тестовая конфигурация, без отказоустойчивости; `offlineMode` относится к лицензированию и сам по себе не отключает все внешние функции.

Проверить список образов после изменения настроек: `helm template budibase ./budibase -n budibase -f local-values.yaml | grep 'image:'`. В него попадут также образы включённых дополнительных компонентов.
