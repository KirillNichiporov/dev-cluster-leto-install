# Подготовка БД для monolith 

Переходим на машину с postgresql (192.168.1.71), на машине подключаемся от пользоваетля postgres и переходим к созданию нового юзера и бд

```bash

su - postgres

```

Теперь создаем пользователя

```bash

createuser --interactive

даем права суперпользователя и задаем имя dev_smartdevices_admin

psql 

задаем пароль для юзера

alter user dev_smartdevices_admin with password '********';

```

создаем БД с таким же названием как пользовательскую

```bash

create database dev_smartdevices_admin;

```

Далее создаем нужную нам БД и назначаем влядельцем нашего нового юзера 

```bash

create database dev_smartdevices_db owner dev_smartdevices_admin;

```

так же надо сосздпть пользователя linux, чтоб могли подключаться из под него

```bash

adduser dev_smartdevices_admin

```

Тепрь разрешим пользователю доступ из вне к БД

Открываем pg_hba.conf

```bash

nano /etc/postgresql/14/main/pg_hba.conf  

прописываем строку

host    all       dev_smartdevices_admin        192.168.1.0/24          md5

пока без ssl(для ssl прописываем hostssl)

```

делаем reload

```bash

systemctl reload postgresql

```

далее подключаемся от клинента под новым пользователем для удобства (datagrep и тд),
там создаем схему внутри БД и назначаем владельцем нового пользователя

```
БД подготовлена

# Обновление монолита

Перейдем к обновлению переменных монолита

Внутри манифеста меняем данные о БД и чистим лишнее, добавляем новое и меняем тег имеджа на test

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: monolith-deployment
  labels:
    app: monolith
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monolith
  template:
    metadata:
      labels:
        app: monolith

    spec:
      containers:
      - name: monolith-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/smartdevice/smartdevice_server:test
        env:
        - name: APP_PUBLIC_URL
          value: "http://mn-dev.smartsafeschool.com"
        - name: SPRING_PROFILES_ACTIVE
          value: "production,xmpp"
        - name: DB_HOST
          value: "db-postgre-dev.smartsafeschool.com"
        - name: DB_USER
          value: "dev_smartdevices_admin"
        - name: DB_PASSWORD
          value: "dhert343Xz4q092"
        - name: DB_SCHEME
          value: "dev_smartdevices_db"
#        - name: DB_PARAMETERS
#          value: "createDatabaseIfNotExist=true&useUnicode=true&characterEncoding=UTF-8&allowPublicKeyRetrieval=true&serverTimezone=UTC&enabledTLSProtocols=TLSv1.2&useSSL=true"

        - name: MAIL_PASSWORD
          value: "kdnT21hdQs"
        - name: MAIL_USERNAME
          value: "dev-notif-mn@smartsafeschool.com"
        - name: MAIL_FROM
          value: "dev-notif-mn@smartsafeschool.com"

        - name: APNS_DEVELOP_SERVER
          value: "false"
        - name: APNS_P12_PASSWORD
          value: "WfYvoRv5QTJOqOW23kD2"
        - name: NETTY_MODULE_TOKEN
          value: "954e1ece-494b-48fd-86c2-58b017795b8a"
        - name: APP_CORS_ALLOWED_ORIGINS
          value: "*"
        - name: APP_CORS_ALLOW_CREDENTIALS
          value: "false"


        - name: APP_XMPP_HOST
          value: "prosody-service.prosody"
        - name: APP_XMPP_XMPP_DOMAIN
          value: "messenger-jitsi-dev.smartsafeschool.com"
        - name: APP_XMPP_USER
          value: "api.server"
        - name: APP_XMPP_PASSWORD
          value: "1cf44b4e337f54270f7b870a90415b8e"
        - name: APP_XMPP_TRUST_ALL_SSL
          value: "true"

        - name: APP_JITSI_XMPP_DOMAIN
          value: "messenger-jitsi-dev.smartsafeschool.com"
        - name: APP_JITSI_JWT_APP_SECRET
          value: "3752143452fc0f705564ef4086f3aab6"

        - name: API_ACTIVATION
          value: "https://admin-dev.smartsafeschool.com/confirm-email"



        ports:
        - containerPort: 8081
          protocol: TCP
        - containerPort: 8090
          protocol: TCP
        - containerPort: 8091
          protocol: TCP
        volumeMounts:
        - name: app-storage
          mountPath: /app/storage
        - name: app-credentials
          mountPath: /app/credentials

      volumes:
      - name: app-storage
        persistentVolumeClaim:
          claimName: monolith-pvc-storage
      - name: app-credentials
        persistentVolumeClaim:
          claimName: monolith-pvc-credentials

      imagePullSecrets:
      - name: registry-cred


```

Обновляем деплоймент(выбираем нужный namespace)

```bash

kubectl apply -f monolith-deployment.yaml -n monolith

```
Внутри k9s также рестратуем деплоймент фронта

смотрим поднялись ли поды 

```bash

kubectl get pods -n monolith

NAME                                      READY   STATUS    RESTARTS   AGE
monolith-deployment-79cdc8867b-bm25m      1/1     Running   0          16h
monolith-web-deployment-89f849d7d-j4vtk   1/1     Running   0          16h

```

Готово













