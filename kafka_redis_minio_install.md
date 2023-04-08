# Подготовка кластера для установки Kafka и Redis

В первую очередь создадим namespace (делаем на машине администратора) 

```bash

kubectl create namespace kafka

kubectl create namespace kafka

```

Переходим на машину nfs, создаем на ней каталоги для хранения

```bash

mkdir /var/nfs/kafka

mkdir /var/nfs/redis

пользователя anon делаем владельцем, дабы кластер мог подключиться потом к каталогам

chown anon /var/nfs/redis

chown anon /var/nfs/kafka

```

Открываем файл exports, прописываем строки внутри и обновляем

```bash

vi /etc/exports

/var/nfs/redis 192.168.1.0/24(rw,sync,all_squash,root_squash,anonuid=1001,anongid=1001)

/var/nfs/kafka 192.168.1.0/24(rw,sync,all_squash,root_squash,anonuid=1001,anongid=1001)

exportfs -a

```

C nfs все, теперь установим provisioner в кластер

Заходим на машину администратора и выполняем команды, внутри команд нужно указать хост, а так же путь к каталогу, что будет использоваться и имя storageclass

```bash

helm install nfs-redis-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.1.80 --set nfs.path=/var/nfs/redis --set storageClass.name=redis-storage

helm install nfs-redis-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.1.80 --set nfs.path=/var/nfs/redis --set storageClass.name=redis-storage

```

после выполнения проверим результат

```bash

kubectl get pods -n default

NAME                                                              READY   STATUS    RESTARTS      AGE
nfs-kafka-subdir-external-provisioner-nfs-subdir-external-wt72v   1/1     Running   2 (18h ago)   24h
nfs-monolith-subdir-external-provisioner-nfs-subdir-extern2prvm   1/1     Running   3 (18h ago)   8d
nfs-redis-subdir-external-provisioner-nfs-subdir-external-4666f   1/1     Running   2 (18h ago)   24h

```

Отлично, все в статусе running, теперь можно переходить к установкам

# Установка Kafka

Устанавливать будем из репозитория bitnami для helm, добавим репозиторий, обновим и установим kafka, указываем storageclass, количество реплик, namespace, остальные параметры можно оставить по умолчанию пока

```bash

helm repo add https://charts.bitnami.com/bitnami

helm repo update

helm install kafka bitnami/kafka --set global.storageClass="kafka-storage" --set replicaCount=3 -n kafka

```

Проверим статус 

```bash

kubectl get pods -n kafka

NAME                        READY   STATUS    RESTARTS      AGE
kafka-0                     1/1     Running   1 (18h ago)   23h
kafka-1                     1/1     Running   1 (18h ago)   23h
kafka-2                     1/1     Running   2 (18h ago)   23h
kafka-zookeeper-0           1/1     Running   1 (18h ago)   23h

```

Теперь поднимем ui

Клонируем в любой удобный каталог репозиторий ui

git clone https://github.com/provectus/kafka-ui.git

Далее внутри в charts редактируем файл values.yaml, нужно раскоментировать строки ниже, указать название кластера kafka и сервис для подключенияl.

```bash

yamlApplicationConfig:
  
  kafka:
    clusters:
      - name: dev-kafka-cluster
        bootstrapServers: kafka:9092
  auth:
    type: disabled
  management:
    health:
      ldap:
        enabled: false

```

Для ингресс с tls везде ставим true и указываем dns с именем секрета с сертификатами, создаем его в namespace kafka

```bash

kubectl create secret tls tls-secret --cert=fullchain.pem --key=privkey.pem -n kafka

```

```bash

ingress:
  # Enable ingress resource
  enabled: true

  # Annotations for the Ingress
  annotations: {}

  # ingressClassName for the Ingress
  ingressClassName: "nginx"

  # The path for the Ingress
  path: "/"

  # The path type for the Ingress
  pathType: "Prefix"  

  # The hostname for the Ingress
  host: "dev-kafka.smartsafeschool.com"

  # configs for Ingress TLS
  tls:
    # Enable TLS termination for the Ingress
    enabled: true
    # the name of a pre-created Secret containing a TLS private key and certificate
    secretName: "tls-secret"

  # HTTP paths to add to the Ingress before the default path
  precedingPaths: []

  # Http paths to add to the Ingress after the default path
  succeedingPaths: []

```

Устанавливаем ui

```bash

helm install kafka-ui charts/kafka-ui -n kafka

```


Проверим статус

```bash

NAME                        READY   STATUS    RESTARTS      AGE
kafka-0                     1/1     Running   1 (18h ago)   24h
kafka-1                     1/1     Running   1 (18h ago)   24h
kafka-2                     1/1     Running   2 (18h ago)   24h
kafka-ui-84d59c47df-bh5sc   1/1     Running   0             17m
kafka-zookeeper-0           1/1     Running   1 (18h ago)   24h

```

Отлично, статус running, остается добавить dns на машину k8s-proxy, установка закончена на этом,

# Установка Redis

Создаем pvc для мастера redis, ибо при установке необходимо указывать хранилку, cоздаем файл формата yaml, а в нем указываем storageclass ресурс и пространство имен

```bash

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-for-master-redis
  namespace: redis
spec:
  storageClassName: "redis-storage"
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi

```

```bash

Namespace указан в манифесте, тогда его можно не указывать в команде

kubectl apply -f pvc-master.yaml

```

Проверяем 

```bash

kubectl get pvc -n redis

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-for-master-redis              Bound    pvc-d3c2e550-1a07-4bf0-ac10-f141d9c221d4   4Gi        RWO            redis-storage   24h

```
Отлично, статус bound

Теперь установим redis через helm, указываем реплики, storageclass и pvc для мастера, pvc для реплик создаются автоматически, размер pvc для реплик

```bash

helm install dev-redis bitnami/redis --namespace redis --set global.redis.password=password,master.persistence.existingClaim=pvc-for-master-redis,replica.replicaCount=2,replica.persistence.storageClass=redis-storage,replica.persistence.size=2Gi

```

Проверим статус

```bash

kubectl get pods -n redis

NAME                           READY   STATUS    RESTARTS      AGE
dev-redis-master-0             1/1     Running   1 (19h ago)   25h
dev-redis-replicas-0           1/1     Running   1 (19h ago)   25h
dev-redis-replicas-1           1/1     Running   1 (19h ago)   25h

```

Redis поднялся, установим для него ui

```bash

Клонируем репозиторийи и заходим по пути k8s/manifests

git clone https://github.com/patrikx3/redis-ui.git

Редактируем манифесты по нас(namespace во всех, ingress и configmap отдельно), в configmap указываем креды к redis и сервис с портом

apiVersion: v1
kind: ConfigMap
metadata:
  name: p3x-redis-ui-settings
  namespace: redis
data:
  .p3xrs-conns.json: |
    {
      "list": [
        {
          "name": "dev-redis-cluster",
          "host": "dev-redis-master",
          "port": 6379,
          "password": "password",
          "id": "unique"
        }
      ],
      "license": ""
    }

```

В ингресс указываем dns и секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: redis-ui-ingress
  namespace: redis
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: dev-redis.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: p3x-redis-ui-service
                port:
                  number: 7843

  tls:
  - hosts:
    - dev-redis.smartsafeschool.com 
    secretName: tls-secret

```

Запускаем все в строгом порядке

```bash

kubectl apply -f configmap.yaml

kubectl apply -f deployment.yaml

kubectl apply -f service.yaml

kubectl apply -f ingress.yaml

```

Проверим статус

```bash

kubectl get pods -n redis

NAME                           READY   STATUS    RESTARTS      AGE
dev-redis-master-0             1/1     Running   1 (19h ago)   25h
dev-redis-replicas-0           1/1     Running   1 (19h ago)   25h
dev-redis-replicas-1           1/1     Running   1 (19h ago)   25h
p3x-redis-ui-647c596c5-bsc5c   1/1     Running   1 (19h ago)   24h

```

Готово, добавляем dns на k8s-proxy, установка закончена

# Установка MinIO

Устанавливаться minio будет вне кластера, поэтому готовим машину dev-minio-vm.smarsafeschool 192.168.1.81, для нее необходимо 3 диска, один системный на 30гб, а 2 других по 60гб (меньше 50 не брать ибо minio не запишет системный файл) под minio, точки монтирования устанавливаем в /minio/disk1 /minio/disk2 соответственно, файловая система рекомендована xfs.


После создания машины приступаем к установке minio

Скачиваем пакет сервиса и устанавливаем его

```bash

wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20230324214123.0.0_amd64.deb -O minio.deb

sudo dpkg -i minio.deb

```

Установится он в /etc/systemd/system/minio.service

Открыв и посмотрев файл можнозаметить, что выполняются действия от пользователя minio-userio-user 



































