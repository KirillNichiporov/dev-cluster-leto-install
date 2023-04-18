# Подготовка к развертыванию smartsafe-store.

В первую очередь создадим namespace (делаем на машине администратора) 

```bash

kubectl create namespace store

```

### Создание секрета с кредами для доступа к gitlab registry

На машине администратора выполняем команду (секрет надо поместить в namespace store иначе приложение его не найдет)

```bash

kubectl create secret -n store docker-registry registry-cred --docker-server=https://git2.uonmap.com:5555 --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>

```
### Создание секрета TLS

На машине администратора выполняем команду (секрет надо поместить в namespace store иначе приложение его не найдет)

```bash

kubectl create secret -n store tls tls-secret --cert=/path/fullchain.pem --key=/path/privkey.pem

```

### Создаем сервис для подключения к postgresql

```bash

kind: Endpoints
apiVersion: v1
metadata:
 name: ext-postgres
subsets:
 - addresses: 
   - ip: 192.168.1.71
   ports:
     - port: 5432
---
kind: Service
apiVersion: v1
metadata:
 name: ext-postgres
spec:
 type: ClusterIP
 ports:
 - port: 5432
   targetPort: 5432
   
```
Отправляем конфигурацию в кластер

```bash

kubectl apply -f postgres.yaml -n store

```

# Развертка auth-server

Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-server-deployment
  labels:
    app: auth-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-server
  template:
    metadata:
      labels:
        app: auth-server

    spec:
      containers:
      - name: auth-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/auth_layer/auth_server:dev-k8s
        env:

        - name: spring.application.name
          value: "auth-server"
        - name: spring.security.user.name
          value: "user"
        - name: spring.security.user.password
          value: "123123"
        - name: server.port
          value: "1443"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"

        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: management.health.redis.enabled
          value: "false"
        - name: eureka.client.enabled
          value: "false"
          

        - name: spring.redis.host
          value: "dev-redis-master.redis"
        - name: spring.redis.port
          value: "6379"
        - name: spring.redis.password
          value: "K17n02*i01"
        - name: network.allowed-origins.additional
          value: "http://localhost:5173,http://localhost:18084, https://manager-dev.smartsafeschool.com, https://b2b-dev.smartsafeschool.com, https://store-dev.smartsafeschool.com"

        - name: config.restfull.security.smart-safe-school-store.server
          value: "store-server-service.store"
        - name: config.restfull.security.smart-safe-school-store.jwt.expiration-time
          value: "60"
        - name: config.restfull.security.smart-safe-school-store.refresh-jwt.expiration-time
          value: "1800"
        - name: config.restfull.message.print-entity-id
          value: "true"

        - name: spring.r2dbc.url
          value: "r2dbc:pool:postgresql://ext-postgres:5432/dev_auth_db"
        - name: spring.r2dbc.username
          value: "dev_auth_admin"
        - name: spring.r2dbc.password
          value: "9eea0179abbc460899a8a9f9a"

        ports:
        - containerPort: 1443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred


```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f auth-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: auth-server-service
spec:
  selector:
    app: auth-server
  ports:
  - name: port1
    protocol: TCP
    port: 1443


```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: auth-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: auth-server-service
                port:
                  number: 1443
  tls:
  - hosts:
    - auth-dev.smartsafeschool.com 
    secretName: tls-secret


```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f auth-service.yaml -n store

kubectl apply -f auth-ingress.yaml -n store

```


На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```
# Развертка product-catalog-server


Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog-server-deployment
  labels:
    app: product-catalog-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-catalog-server
  template:
    metadata:
      labels:
        app: product-catalog-server

    spec:
      containers:
      - name: product-catalog-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/product_catalog_layer/product_catalog_server:dev-k8s
        env:

        - name: spring.application.name
          value: "product-catalog"
        - name: java.awt.headless
          value: "true"
        - name: spring.security.user.name
          value: "user"
        - name: spring.security.user.password
          value: "123123"
        - name: server.port
          value: "5443"
        - name: server.ssl.enabled
          value: "true"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"

        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: management.health.redis.enabled
          value: "false"
        - name: eureka.client.enabled
          value: "false"

        - name: spring.redis.host
          value: "dev-redis-master.redis"
        - name: spring.redis.port
          value: "6379"
        - name: spring.redis.password
          value: "K17n02*i01"
        - name: network.allowed-origins.additional
          value: "http://localhost:5173,http://localhost:18084, https://manager-dev.smartsafeschool.com, https://b2b-dev.smartsafeschool.com, https://store-dev.smartsafeschool.com"

        - name: config.security.bucket4j.enabled
          value: "true"

        - name: spring.r2dbc.url
          value: "r2dbc:pool:postgresql://ext-postgres:5432/dev_pc_catalog_db"
        - name: spring.r2dbc.username
          value: "dev_pc_catalog_admin"
        - name: spring.r2dbc.password
          value: "869020cc1e5b4330bb1762a15"


        ports:
        - containerPort: 5443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f product-catalog-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: product-catalog-server-service
spec:
  selector:
    app: product-catalog-server
  ports:
  - name: port1
    protocol: TCP
    port: 5443

```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: product-catalog-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: product-catalog-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: product-catalog-server-service
                port:
                  number: 5443
  tls:
  - hosts:
    - product-catalog-dev.smartsafeschool.com 
    secretName: tls-secret

```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f product-catalog-service.yaml -n store

kubectl apply -f product-catalog-ingress.yaml -n store

```


На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```

# Развертка warehouse-server

Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: warehouse-server-deployment
  labels:
    app: warehouse-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: warehouse-server
  template:
    metadata:
      labels:
        app: warehouse-server

    spec:
      containers:
      - name: warehouse-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/ware_house_layer/ware_house_server:dev-k8s
        env:

        - name: spring.application.name
          value: "warehouse"
        - name: java.awt.headless
          value: "true"
        - name: spring.security.user.name
          value: "user"
        - name: spring.security.user.password
          value: "123123"
        - name: server.port
          value: "9443"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"

        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: management.health.redis.enabled
          value: "false"
        - name: eureka.client.enabled
          value: "false"

        - name: spring.redis.host
          value: "dev-redis-master.redis"
        - name: spring.redis.port
          value: "6379"
        - name: spring.redis.password
          value: "K17n02*i01"
        - name: network.allowed-origins.additional
          value: "http://localhost:5173,http://localhost:18084, https://manager-dev.smartsafeschool.com, https://b2b-dev.smartsafeschool.com, https://store-dev.smartsafeschool.com"

        - name: config.security.bucket4j.enabled
          value: "true"

        - name: spring.r2dbc.url
          value: "r2dbc:pool:postgresql://ext-postgres:5432/dev_wh_warehouse_db"
        - name: spring.r2dbc.username
          value: "dev_wh_warehouse_admin"
        - name: spring.r2dbc.password
          value: "e5e18cee013947b997f0fa63d"

        ports:
        - containerPort: 9443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f warehouse-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: warehouse-server-service
spec:
  selector:
    app: warehouse-server
  ports:
  - name: port1
    protocol: TCP
    port: 9443

```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: warehouse-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: warehouse-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: warehouse-server-service
                port:
                  number: 9443
  tls:
  - hosts:
    - warehouse-dev.smartsafeschool.com 
    secretName: tls-secret


```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f warehouse-service.yaml -n store

kubectl apply -f warehouse-ingress.yaml -n store

```


На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```

# Развертка email-server

Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mail-server-deployment
  labels:
    app: mail-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mail-server
  template:
    metadata:
      labels:
        app: mail-server

    spec:
      containers:
      - name: mail-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/notif_layer/email_server:dev-k8s
        env:

        - name: spring.application.name
          value: "smart-safe-school-mail"
        - name: server.port
          value: "2443"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"
        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: eureka.client.enabled
          value: "false"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: config.restfull.message.print-entity-id
          value: "true"

        ports:
        - containerPort: 2443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred


```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f mail-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: mail-server-service
spec:
  selector:
    app: mail-server
  ports:
  - name: port1
    protocol: TCP
    port: 2443


```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mail-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: mail-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mail-server-service
                port:
                  number: 2443
  tls:
  - hosts:
    - mail-dev.smartsafeschool.com 
    secretName: tls-secret

```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f mail-service.yaml -n store

kubectl apply -f mail-ingress.yaml -n store

```


На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```

# Развертка media-server


Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: media-server-deployment
  labels:
    app: media-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: media-server
  template:
    metadata:
      labels:
        app: media-server

    spec:
      containers:
      - name: media-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/media_layer/media_server:dev-k8s
        env:

        - name: spring.application.name
          value: "smart-safe-school-media"
        - name: server.port
          value: "4443"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"

        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"

        - name: eureka.client.enabled
          value: "false"

        - name: config.filestorage.host
          value: "https://dev-minio-api.smartsafeschool.com"

        - name: network.allowed-origins.additional
          value: "http://localhost:5173,http://localhost:18084, https://manager-dev.smartsafeschool.com, https://b2b-dev.smartsafeschool.com, https://store-dev.smartsafeschool.com"



        ports:
        - containerPort: 4443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred


```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f media-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: media-server-service
spec:
  selector:
    app: media-server
  ports:
  - name: port1
    protocol: TCP
    port: 4443

```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: media-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: media-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: media-server-service
                port:
                  number: 4443
  tls:
  - hosts:
    - media-dev.smartsafeschool.com 
    secretName: tls-secret

```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f media-service.yaml -n store

kubectl apply -f media-ingress.yaml -n store

```

На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```

# Развертка payment-server


Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: pmt-server-deployment
  labels:
    app: pmt-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pmt-server
  template:
    metadata:
      labels:
        app: pmt-server

    spec:
      containers:
      - name: pmt-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/payment_layer/payment_server:dev-k8s
        env:

        - name: spring.application.name
          value: "smart-safe-school-pmt"
        - name: server.port
          value: "7443"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"

        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: eureka.client.enabled
          value: "false"

        - name: config.restfull.message.print-entity-id
          value: "true"


        ports:
        - containerPort: 7443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred


```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f pmt-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: pmt-server-service
spec:
  selector:
    app: pmt-server
  ports:
  - name: port1
    protocol: TCP
    port: 7443


```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pmt-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: pmt-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pmt-server-service
                port:
                  number: 7443
  tls:
  - hosts:
    - pmt-dev.smartsafeschool.com 
    secretName: tls-secret


```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f pmt-service.yaml -n store

kubectl apply -f pmt-ingress.yaml -n store

```

На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```

# Развертка store-server

Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: store-server-deployment
  labels:
    app: store-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: store-server
  template:
    metadata:
      labels:
        app: store-server

    spec:
      containers:
      - name: store-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/store_layer/store_server:dev-k8s
        env:

        - name: spring.application.name
          value: "smart-safe-school-store"
        - name: java.awt.headless
          value: "true"
        - name: spring.security.user.name
          value: "user"
        - name: spring.security.user.password
          value: "123123"
        - name: server.port
          value: "8443"
        - name: server.ssl.enabled
          value: "true"
        - name: server.ssl.key-store
          value: "classpath:keystore.jks"

        - name: server.ssl.key-store-password
          value: "123123-minsk"
        - name: log4j.debug
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: management.health.redis.enabled
          value: "false"
        - name: eureka.client.enabled
          value: "false"

        - name: spring.redis.host
          value: "dev-redis-master.redis"
        - name: spring.redis.port
          value: "6379"
        - name: spring.redis.password
          value: "K17n02*i01"
        - name: network.allowed-origins.additional
          value: "http://localhost:5173,http://localhost:18084, https://manager-dev.smartsafeschool.com, https://b2b-dev.smartsafeschool.com, https://store-dev.smartsafeschool.com"

        - name: config.security.bucket4j.enabled
          value: "true"

        - name: spring.r2dbc.url
          value: "r2dbc:pool:postgresql://ext-postgres:5432/dev_sss_store_db"
        - name: spring.r2dbc.username
          value: "dev_sss_store_admin"
        - name: spring.r2dbc.password
          value: "d8c481caab82442bb772cd893"

        - name: config.restfull.security.product-catalog.server.host-port
          value: "product-catalog-server-service.store:5443"
        - name: config.restfull.security.warehouse.server.host-port
          value: "warehouse-server-service.store:9443"
        - name: config.ui.host
          value: "https://b2b-dev.smartsafeschool.com"


        ports:
        - containerPort: 8443
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

Теперь выполняем команду для деплоя auth-server (запускаем в нужном namespace)

```bash

kubectl apply -f store-deployment.yaml -n store

```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: store-server-service
spec:
  selector:
    app: store-server
  ports:
  - name: port1
    protocol: TCP
    port: 8443


```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-server-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: store-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: store-server-service
                port:
                  number: 8443
  tls:
  - hosts:
    - store-dev.smartsafeschool.com 
    secretName: tls-secret

```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f store-service.yaml -n store

kubectl apply -f store-ingress.yaml -n store

```

На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```



# Проверка

Проверим статус всех сервисов 

```bash

kubectl get pods -n store

NAME                                                 READY   STATUS    RESTARTS         AGE
auth-server-deployment-7bb449f64b-t9nrx              1/1     Running   0                38m
mail-server-deployment-5d85c78c-754p2                1/1     Running   1 (12h ago)      3d19h
media-server-deployment-557cbd7d56-m8zsq             1/1     Running   92 (4h26m ago)   19h
product-catalog-server-deployment-67884886cb-jl9fq   1/1     Running   1 (12h ago)      18h
store-server-deployment-6c4fc78957-crmjh             1/1     Running   1 (12h ago)      18h
warehouse-server-deployment-5fddf8b947-sch4d         1/1     Running   1 (12h ago)      20h

```

Отлично, все в статусе running, на этом деплой закончен


















