# Подготовка БД

### PostgreSQL

Для сервисов Actors и Moodle-auth создаем пользователей и бд.

Выполняем на машине с PostgreSQL

```bash

su - postgres

psql

create user YOUR_USER with password 'PASSWORD';

create database YOUR_DB with owner YOUR_USER;

```

Далее в файле /etc/postgresql/16/main/pg_hba.conf нужно прописать правила удаленного доступа для созданных пользователей

БД проинициализировать можно через DataGrip


Для Lms создаем пользователя и даем ему права только на чтение БД moodle-external

```bash

su - postgres

psql

create user YOUR_USER with password 'PASSWORD';

grant connect on database MOODLE_DB to YOUR_USER;

коннектимся к БД сервиса moodle-external

\c MOODLE_DB

Выдаем права пользователю на использование схемы public и всех таблиц

grant usage on schema public to YOUR_USER;

grant select on all tables in schema public to YOUR_USER;

```

### MongoDB

На машине с mongo подключаемся к mongosh используя свой пароль(на деве это, а на тесте другой вроде). Также если есть tls то указываем флаг

```bash


mongosh -u admin-user -p K17n02*i01 --host localhost.smartsafeschool.com --tls 

Далее

use admin

use YOUR_DB

B прописываем команду заменив бд и пользователя, и пароль

db.createUser({ user: "dev_lms_admin",pwd: "beacd541552e4e18ba4b693f6",roles:[ {role: "readWrite",db: "dev_lms_db"} ],mechanisms:["SCRAM-SHA-1"]})

```

С базами закончили



# Деплой сервисов

Прежде чем деплоить сервисы нужно вытянуть токен из moodle-external

Переходим на dev-moodle.smartsafeschool.com(в моем случае) и заходим под админом.

Далее переходи в вкдладку

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/42a27f1b-6d62-4aed-a8cb-37177ed77c14)


![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/9ac314b5-ad4d-497d-8256-ca9b060b276e)

Ставим Enabled и сохраняем иначе не появится токен админа и мы не сможем использовать вебсервисы

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/aa6ea214-2fa4-4287-8543-0ba2d8abaa93)


Далее включим внешний сервис


В вкладке Site Administration -> Server


![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/09521354-bd08-4401-9df6-89c4e17adcff)


![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/b670004d-4502-40b9-91ed-7bccd4a889de)


Ставим Enabled и сохраняем


![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/6bf61f1b-4ac9-4177-a32c-d36db479f081)


Теперь вытянем токен 

```bash

 curl -X POST "https://dev-moodle.smartsafeschool.com/login/token.php"      -d "username=admin"      -d "password=adminadmin"      -d "service=moodle_mobile_app"

```

 или

В вкладке Site Administration -> Server

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/22a16fec-4ab8-46d9-9e90-71ea897e993f)
 

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/11a50e61-b47d-4af9-9597-5d1a117ba22c)


Токен надо добаввить в moodle-server и lms-server в переменные

НЕ ЗАБЫВАЕМ РЕДАКТИРОВАТЬ ВСЕ ПЕРЕМЕННЫЕ ПОД СВОИ НУЖДЫ)

### Actors 


######  Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: actors-server-deployment
  labels:
    app: actors-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: actors-server
  template:
    metadata:
      labels:
        app: actors-server
    spec:
      containers:
      - name: actors-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/sss_services/3s-actors/actors:dev-latest
        imagePullPolicy: Always
        env:
        - name: spring.profiles.active
          value: "k9-dev"
        - name: log4j.debug
          value: "true"
        - name: server.k9sMode
          value: "true"
        - name: spring.r2dbc.url
          value: "r2dbc:postgresql://dev-db-pg.smartsafeschool.com:5432/dev_actors_db"
        - name: spring.r2dbc.username
          value: "dev_actors_admin"
        - name: spring.r2dbc.password
          value: "5686d1b5810245b28c704b8e6"
        - name: management.health.redis.enabled
          value: "true"
        - name: spring.data.redis.host
          value: "dev-redis-master.redis"
        - name: spring.data.redis.port
          value: "6379"
        - name: spring.data.redis.password
          value: "K17n02*i01"
        - name: network.cors.allowed-origins.additional
          value: "https://dev-manager-store.smartsafeschool.com, https://dev-b2b-store.smartsafeschool.com, https://dev-store.smartsafeschool.com, https://dev-account.smartsafeschool.com,https://store.ubuntu1.smartsafeschool.com,https://account.ubuntu3.smartsafeschool.com,https://b2b.ubuntu3.smartsafeschool.com,https://manager.ubuntu3.smartsafeschool.com,https://account.ubuntu6.smartsafeschool.com,https://store.ubuntu6.smartsafeschool.com,https://b2b.ubuntu6.smartsafeschool.com,https://manager.ubuntu6.smartsafeschool.com"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"

        ports:
        - containerPort: 6443
          protocol: TCP
      imagePullSecrets:
      - name: registry-cred


```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: actors-server-service
spec:
  selector:
    app: actors-server
  ports:
  - name: port1
    protocol: TCP
    port: 6443


```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: actors-server-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;

#    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
spec:
  rules:
    - host: dev-actor-api.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: actors-server-service
                port:
                  number: 6443
  tls:
  - hosts:
    - dev-actor-api.smartsafeschool.com
    secretName: tls-secret


```




###  Moodle-auth


######  Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: moodle-auth-server-deployment
  labels:
    app: moodle-auth-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moodle-auth-server
  template:
    metadata:
      labels:
        app: moodle-auth-server
    spec:
      containers:
      - name: moodle-auth-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/sss_services/3s-moodle/moodle-auth:dev-latest
        imagePullPolicy: Always
        env:
        - name: spring.profiles.active
          value: "k9-dev"
        - name: log4j.debug
          value: "true"
        - name: server.k9sMode
          value: "true"
        - name: spring.r2dbc.url
          value: "r2dbc:postgresql://dev-db-pg.smartsafeschool.com:5432/dev_moodle_auth_db"
        - name: spring.r2dbc.username
          value: "dev_moodle_auth_admin"
        - name: spring.r2dbc.password
          value: "7045d90d6b3946688e9f8ef55"
        - name: management.health.redis.enabled
          value: "true"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"

        ports:
        - containerPort: 3445
          protocol: TCP
      imagePullSecrets:
      - name: registry-cred


```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: moodle-auth-server-service
spec:
  selector:
    app: moodle-auth-server
  ports:
  - name: port1
    protocol: TCP
    port: 3445


```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: moodle-auth-server-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;

#    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
spec:
  rules:
    - host: dev-moodle-auth-api.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dev-moodle-auth-server-service
                port:
                  number: 3445
  tls:
  - hosts:
    - dev-moodle-auth-server-api.smartsafeschool.com
    secretName: tls-secret


```



### Moodle-server


######  Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: moodle-server-integration-deployment
  labels:
    app: moodle-server-integration
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moodle-server-integration
  template:
    metadata:
      labels:
        app: moodle-server-integration
    spec:
      containers:
      - name: moodle-server-integration-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/sss_services/3s-moodle/moodle:dev-latest
        imagePullPolicy: Always
        env:
        - name: spring.profiles.active
          value: "k9-dev"
        - name: log4j.debug
          value: "true"
        - name: server.k9sMode
          value: "true"
        - name: spring.r2dbc.url
          value: "r2dbc:postgresql://dev-db-pg.smartsafeschool.com:5432/moodle_db"
        - name: spring.r2dbc.username
          value: "dev_moodle_server"
        - name: spring.r2dbc.password
          value: "fe2561aa9b9041d8add579dce"
        - name: management.health.redis.enabled
          value: "true"
        - name: network.cors.allowed-origins.additional
          value: "https://dev-manager-store.smartsafeschool.com, https://dev-b2b-store.smartsafeschool.com, https://dev-store.smartsafeschool.com, https://dev-account.smartsafeschool.com,https://store.ubuntu1.smartsafeschool.com,https://account.ubuntu3.smartsafeschool.com,https://b2b.ubuntu3.smartsafeschool.com,https://manager.ubuntu3.smartsafeschool.com,https://account.ubuntu6.smartsafeschool.com,https://store.ubuntu6.smartsafeschool.com,https://b2b.ubuntu6.smartsafeschool.com,https://manager.ubuntu6.smartsafeschool.com"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: lms.moodle.enabled
          value: "true"
        - name: lms.moodle.token
          value: "9357a68a6440c518c31e341c8083938c"
        - name: lms.moodle.host
          value: "http://moodle-external-service.lms:8080"


        ports:
        - containerPort: 3444
          protocol: TCP
        - containerPort: 13444
          protocol: TCP
      imagePullSecrets:
      - name: registry-cred


```

###### Service

```bash

---
apiVersion: v1
kind: Service
metadata:
  name: moodle-server-integration-service
spec:
  selector:
    app: moodle-server-integration
  ports:
  - name: port1
    protocol: TCP
    port: 3444

---
apiVersion: v1
kind: Service
metadata:
  name: moodle-server-integration-grpc-service
spec:
  selector:
    app: moodle-server-integration
  ports:
  - name: port1
    protocol: TCP
    port: 13444


```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: moodle-server-integration-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;

#    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
spec:
  rules:
    - host: dev-moodle-server-integration-api.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: moodle-server-integration-service
                port:
                  number: 3444
  tls:
  - hosts:
    - dev-moodle-server-integration-api.smartsafeschool.com
    secretName: tls-secret

```


### Lms-server


######  Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: lms-server-deployment
  labels:
    app: lms-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lms-server
  template:
    metadata:
      labels:
        app: lms-server
    spec:
      containers:
      - name: lms-server-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/sss_services/3s-lms/lms:dev-latest
        imagePullPolicy: Always
        env:
        - name: spring.profiles.active
          value: "k9-dev"
        - name: log4j.debug
          value: "true"
        - name: server.k9sMode
          value: "true"
        - name: spring.data.mongodb.uri
          value: "mongodb://dev_lms_admin:beacd541552e4e18ba4b693f6@dev-db-mongo.smartsafeschool.com:27017/dev_lms_db?tls=true"
        - name: lms.moodle.enabled
          value: "true"
        - name: lms.moodle.token
          value: "9357a68a6440c518c31e341c8083938c"
        - name: lms.moodle.host
          value: "http://moodle-external-service.lms:8080"
        - name: management.health.redis.enabled
          value: "true"
        - name: network.cors.allowed-origins.additional
          value: "https://dev-manager-store.smartsafeschool.com, https://dev-b2b-store.smartsafeschool.com, https://dev-store.smartsafeschool.com, https://dev-account.smartsafeschool.com,https://store.ubuntu1.smartsafeschool.com,https://account.ubuntu3.smartsafeschool.com,https://b2b.ubuntu3.smartsafeschool.com,https://manager.ubuntu3.smartsafeschool.com,https://account.ubuntu6.smartsafeschool.com,https://store.ubuntu6.smartsafeschool.com,https://b2b.ubuntu6.smartsafeschool.com,https://manager.ubuntu6.smartsafeschool.com"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"
        - name: config.grpc.moodle-service.server.host
          value: "moodle-server-integration-grpc-service.lms"
        - name: config.grpc.moodle-service.server.port
          value: "13444"

        ports:
        - containerPort: 3443
          protocol: TCP
      imagePullSecrets:
      - name: registry-cred


```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: lms-server-service
spec:
  selector:
    app: lms-server
  ports:
  - name: port1
    protocol: TCP
    port: 3443


```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lms-server-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;

#    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    
spec:
  rules:
    - host: dev-lms-server-api.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: lms-server-service
                port:
                  number: 3443
  tls:
  - hosts:
    - dev-lms-server-api.smartsafeschool.com
    secretName: tls-secret

```

Для деплоя каждого манифеста выполняем команду(не забываем указать нужный неймспейс)

```bash

kubectl apply -f YOUR_YAML_FILE -n NAMESPACE

проверить сервисы можно командой

kubectl get pods -n NAMESPACE

```

Далее записи 

```bash

dev-lms-api.smartsafeschool.com
dev-actor-api.smartsafeschool.com

```

Добавляем на прокси машину  и делаем

```bash

nginx -t
nginx -s reload

```

