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

Отлично, статус running, установка закончена на этом

# Установка Redis

Создаем pvc для мастера redis, ибо при установке необходимо указывать хранилку, cоздаем файл формата yaml

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
Отлично

























































