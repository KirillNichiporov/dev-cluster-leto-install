# Подготовка БД
На машине 192.168.1.71 выполняем команды по настройке пользователя и новой бд

```bash

su - postgres

psql

create user moodle with password 'your_pass';

create database moodle_db with owner moodle;

```
БД и пользователь созданы, можно проверить командой 

```bash

\l

          Name           |         Owner          | Encoding |   Collate   |    Ctype    |                 Access privileges                 
-------------------------+------------------------+----------+-------------+-------------+---------------------------------------------------
 admin                   | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_auth_admin          | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_auth_db             | dev_auth_admin         | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_pc_catalog_admin    | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_pc_catalog_db       | dev_pc_catalog_admin   | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_prosody_admin       | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_prosody_db          | dev_prosody_admin      | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_smartdevices_admin  | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_smartdevices_db     | dev_smartdevices_admin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/dev_smartdevices_admin                       +
                         |                        |          |             |             | dev_smartdevices_admin=CTc/dev_smartdevices_admin+
                         |                        |          |             |             | smart_ro=c/dev_smartdevices_admin
 dev_sss_store_admin     | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_sss_store_db        | dev_sss_store_admin    | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_wh_warehouse_admin  | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 dev_wh_warehouse_db     | dev_wh_warehouse_admin | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 moodle_db               | moodle                 | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres                | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template1               | postgres               | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres                                      +
                         |                        |          |             |             | postgres=CTc/postgres
 test_user_long_database | admin                  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
(17 rows)


```

Видим, что все создано, как нам небходимо, пользователь владелец бд

Далее создадим пользователя и дадим ему доступ к бд монолита к определенной таблице только на чтение

```bash

create user smart_ro with password 'GK397Nxs45L78';

grant connect on database dev_smardevices_db to smart_ro;

\connect dev_smartdevices_db

обязательно коннектимся к базе монолита

grant usage on schema dev_smartdevices_schema to smart_ro;

grant select on users to smart_ro;

```

Пермишены выданы, осталось открыть доступ из вне для всех юзеров

В файле /etc/postgresql/14/main/pg_hba.conf прописываем строки 

```bash

host	all		moodle			 192.168.1.0/24	md5
host	all		smart_ro		 192.168.1.0/24 md5

```

После выполняем команду 

```bash

systemctl reload postgresql

```

БД настроена


# Деплой Moodle

Подготовим манифесты

###### Deployment 

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: moodle-deployment
  labels:
    app: moodle
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moodle
  template:
    metadata:
      labels:
        app: moodle

    spec:
      containers:
      - name: moodle-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/third_party_services/moodle:dev-latest
        imagePullPolicy: Always
        env:

        - name: MOODLE_HOST
          value: "moodle-dev.smartsafeschool.com"
        - name: MOODLE_USERNAME
          value: "admin"
        - name: MOODLE_PASSWORD
          value: "adminadmin"
        - name: MOODLE_SITE_NAME
          value: "SSS"
        - name: MOODLE_EMAIL
          value: "moodle@mail.smartsafeschool.com"
        - name: MOODLE_SKIP_BOOTSTRAP
          value: "yes"
#        - name: MOODLE_REVERSEPROXY
#          value: "yes"
#        - name: MOODLE_SSLPROXY
#          value: "true"
        - name: MOODLE_DISABLE_LOGIN_TOKEN
          value: "yes"

        - name: MOODLE_DATABASE_TYPE
          value: "pgsql"
        - name: MOODLE_DATABASE_HOST
          value: "db-postgre-dev.smartsafeschool.com"
        - name: MOODLE_DATABASE_PORT_NUMBER
          value: "5432"
        - name: MOODLE_DATABASE_NAME
          value: "moodle_db"
        - name: MOODLE_DATABASE_USER
          value: "moodle"
        - name: MOODLE_DATABASE_PASSWORD
          value: "8437veRT34zxp2"



        ports:
        - containerPort: 8080
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: moodle-ingress
  namespace: moodle
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: moodle-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: moodle-service
                port:
                  number: 8080
  tls:
  - hosts:
    - moodle-dev.smartsafeschool.com 
    secretName: tls-secret

```

###### Service 

```bash

apiVersion: v1
kind: Service
metadata:
  name: moodle-service
spec:
  selector:
    app: moodle 
  ports:
  - name: port1
    protocol: TCP
    port: 8080

```

Далее выполняем команды по созданию секретов для сертификатов и доступа к регистри

```bash

kubectl create namespace moodle

Не забываем указать правильный путь к сертификатам

kubectl create secret tls tls-secret --cert=fullchain.pem --key=privkey.pem -n moodle

kubectl create secret -n moodle docker-registry registry-cred --docker-server=https://git2.uonmap.com:5555 --docker-username=user_service --docker-password=BELpzoxx_7vsYRpUKCWp --docker-email=<your-email>

kubectl apply -f moodle-deployment.yaml -n moodle

kubectl apply -f moodle-ingress.yaml -n moodle

kubectl apply -f moodle-service.yaml -n moodle

```

Проверим контейнер 

```bash

kubectl get pods -n moodle

NAME                                 READY   STATUS    RESTARTS   AGE
moodle-deployment-58d5b4bcf5-mrd5m   1/1     Running   0          19h

```

Под стартанул, отлично, деплой завершен
