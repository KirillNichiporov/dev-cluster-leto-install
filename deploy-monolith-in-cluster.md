# Подготовка к развертыванию monolith.

Необходимо на рабочих машинах (worker) добавить сертификат в доверенные и в папку /etc/docker иначе имедж приложения не сможет стянуться в кластер.

- Docker Image - git2.uonmap.com:5555/smartsafeschool/backend/smartdevice/smartdevice_server:latest

Также добавляем пользователя user (кластер настраивался через него) в группу docker и приминяем изменения для групп.

```bash

usermod -aG docker user

newgrp docker

```


Добавляем сертификат ca.crt в каталог  /usr/local/share/ca-certificates/ а также /etc/docker/certs.d/git2.uonmap.com:5555/ (недостающие каталоги создать), делаем рестарт сервисов 

```bash

Обновляем сертификаты

update-ca-certificates

Рестартуем сервисы

systemctl restart docker

systemctl restart kubelet

systemctl restart containerd

```


Теперь создадим сервис в кластере для подключения к БД (для манифестов создаем файлы yaml):

```bash

kind: Endpoints
apiVersion: v1
metadata:
 name: ext-mariadb 
subsets:
 - addresses:
     - ip: 192.168.1.70
   ports:
     - port: 3306
---
kind: Service
apiVersion: v1
metadata:
 name: ext-mariadb
spec:
 type: ClusterIP
 ports:
 - port: 3306
   targetPort: 3306

```

Он будет передавать адрес хоста с БД на сервис monolith

Отправим конфигурацию в кластер в namespace monolith (команды по обновлению надо выполнять из каталога, где создавались yaml файлы)

kubectl apply -f mariadb-connect-service.yaml -n monolith

Далее создадим pvc для привязки к nfs 

###### monolith-pvc-credentials

```bash

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: monolith-pvc-credentials
  annotations:
    volume.beta.kubernetes.io/storage-class: "monolith-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
      
```

      
###### monolith-pvc-storage

```bash

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: monolith-pvc-storage
  annotations:
    volume.beta.kubernetes.io/storage-class: "monolith-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 40Gi
      
```

Отправим конфигурацию в кластер в namespace monolith

```bash

kubectl apply -f monolith-pvc-storage.yaml -n monolith

kubectl apply -f monolith-pvc-credentials.yaml -n monolith

```

У pvc должен быть статус Bound

```bash

kubectl get pvc -n monolith

NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
monolith-pvc-credentials   Bound    pvc-2bc6a8e8-7c40-4aa2-b518-c64a520c8895   10Gi       RWX            monolith-storage   2d1h
monolith-pvc-storage       Bound    pvc-e6422bd2-f346-4424-89ea-e2d04d55d1ea   40Gi       RWX            monolith-storage   2d1h

```

### Создание пользователя и БД для monolith в MariaDB

Перемещаемся на машину c maria и подключаемся к mariadb

```bash

mysql -U root -p

Внутри создаем пользователя и БД

CREATE USER 'smart_devices'@'192.168.1.%' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON smartdevices.* TO 'smart_devices'@'192.168.1.%';

```

Данные команды настраивают пользователю доступ не по ssl, чтобы дать доступ именно по ssl нужно дописать в конце REQUIRE SSL

Пользователь и БД готовы


### Создание секрета с кредами для доступа к gitlab registry

На машине администратора выполняем команду (секрет надо поместить в namespace monolith иначе приложение его не найдет)

```bash

kubectl create secret -n monolith docker-registry registry-cred --docker-server=https://git2.uonmap.com:5555 --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>

```
### Создание секрета TLS

На машине администратора выполняем команду (секрет надо поместить в namespace monolith иначе приложение его не найдет)

```bash

kubectl create secret -n monolith tls tls-secret --cert=/path/fullchain.pem --key=/path/privkey.pem

```

# Развертка monolith

Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, доступ к БД передается через ранее созданный сервис локальные папки поделючены к nfs через pvc, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

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
        image: git2.uonmap.com:5555/smartsafeschool/backend/smartdevice/smartdevice_server:latest
        env:
        - name: APP_NAME
          value: "SmartServer"
        - name: APP_PUBLIC_URL
          value: "http://mn-dev.smartsafeschool.com"
        - name: SPRING_PROFILES_ACTIVE
          value: "production"


        - name: DB_HOST
          value: "ext-mariadb"
        - name: DB_USER
          value: "smart_devices"
        - name: DB_PASSWORD
          value: "secret"
        - name: DB_SCHEME
          value: "smartdevices"

        - name: MAIL_HOST
          value: "mail.pm.uonmap.com"
        - name: MAIL_PORT
          value: "587"
        - name: MAIL_PASSWORD
          value: "secret"
        - name: MAIL_USERNAME
          value: "smartsave@pm.uonmap.com"
        - name: MAIL_FROM
          value: "no-reply@smartsafeschool.com"

        - name: APNS_DEVELOP_SERVER
          value: "false"
        - name: APNS_TOPIC
          value: "com.uonmap.SmartSafeSchool"
        - name: APNS_P12_PASSWORD
          value: "secret"
        - name: NETTY_MODULE_TOKEN
          value: "secret"
        - name: APP_CORS_ALLOW_CREDENTIALS
          value: "true"
        - name: APP_CORS_ALLOWED_ORIGINS
          value: "https://admin.mn-dev.smartsafeschool.com"

        - name: ACTIVEMQ_BROKER_URL
          value: "tcp://192.168.1.38:61616"
        - name: ACTIVEMQ_PASSWORD
          value: "secret"
        - name: ACTIVEMQ_USER
          value: "smartdevice_admin"

        - name: APP_ACTIVEMQ_EARTHQUAKE_URI
          value: "activemq:topic:smartserver.earthquake"
        - name: APP_XMPP_HOST
          value: "192.168.1.38"
        - name: APP_XMPP_XMPP_DOMAIN
          value: "mn-dev.smartsafeschool.com"
        - name: APP_XMPP_USER
          value: "api.server"
        - name: APP_XMPP_PASSWORD
          value: "secret"
        - name: APP_XMPP_AUTH
          value: "auth"
        - name: APP_XMPP_MUC
          value: "muc"
        - name: APP_XMPP_TRUST_ALL_SSL
          value: "true"

        - name: APP_JITSI_XMPP_DOMAIN
          value: "mn-dev.smartsafeschool.com"
        - name: APP_JITSI_JWT_APP_ID
          value: "smart_safe_school_app"
        - name: APP_JITSI_JWT_APP_SECRET
          value: "secret"
        - name: APP_JITSI_JWT_ACCEPTED_ISSUERS
          value: "smart_safe_school_app"
        - name: APP_JITSI_JWT_ACCEPTED_AUDIENCES
          value: "smart_safe_school_app"
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

Теперь выполняем команду для деплоя monolith (запускаем в нужном namespace)

```bash

kubectl apply -f monolith-deployment.yaml -n monolith

статус должен быть running

kubectl get pods -n monolith

NAME                                   READY   STATUS    RESTARTS   AGE
monolith-deployment-695f5c8854-prc66   1/1     Running   0          46h

```


Далее необходимо настроить сервис и ингресс для подключения к monolith

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: monolith-service
spec:
  selector:
    app: monolith 
  ports:
  - name: http-rest-api
    protocol: TCP
    port: 8081
  - name: websocket-1
    protocol: TCP
    port: 8090
  - name: websocket-2
    protocol: TCP
    port: 9091

```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monolith-ingress
  namespace: monolith
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: mn-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monolith-service
                port:
                  number: 8081
  tls:
  - hosts:
    - mn-dev.smartsafeschool.com 
    secretName: tls-secret

```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f monolith-service.yaml -n monolith

kubectl apply -f monolith-ingress.yaml -n monolith

```


На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ создаем файл monolith и добавляем в него запись


```bash

server {

        listen 443 ssl;
        server_name mn-dev.smartsafeschool.com;

        ssl_certificate /etc/nginx/certs/fullchain.pem;
        ssl_certificate_key /etc/nginx/certs/privkey.pem;

        location / {
                proxy_set_header Host $host;
                proxy_pass       https://192.168.1.77:30002;

        }


        location /api/ {
                proxy_set_header Host $host;
                proxy_pass       https://192.168.1.77:30002/api/;

        }


        location /ws/ {
                proxy_set_header Host $host;
                proxy_pass       https://192.168.1.77:30002/ws/;

        }


        location /actuator/ {
                proxy_set_header Host $host;
                proxy_pass       https://192.168.1.77:30002/actuator/;

        }
}

```

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```

Проверяем ответ от сервиса

```bash

curl https://mn-dev.smartsafeschool.com/api/admin/version/
{"status":401,"message":null,"error":"Full authentication is required to access this resource","timestamp":"2023-04-05T11:34:41.938Z"}

```

Развертка на этом закончена

# Monolith and MinIO

Редактируем манифесты добавляя новые переменные( пересаживаем на minio, media-server успользуется из пространства имен store

В манифест монолита, строчки связанные с volume storage можно закомментировать

```bash

        - name: APP_CONFIG_CHANGE_EMAIL_DELAY_SECONDS
          value: "90"
        - name: APP_CONFIG_MEDIA_HOST
          value: "http://media-server-service.store"
        - name: APP_CONFIG_MEDIA_PORT
          value: "4443"

```

В манифест веб-сервиса

```bash

        - name: VITE_MEDIA_SERVICE_URL
          value: "https://dev-minio-api.smartsafeschool.com"


```

После редактирования манифестов можно обновить сервисы

```bash

kubectl apply -f monolith-web-deployment.yaml -n monolith
kubectl apply -f monolith-deployment.yaml -n monolith

```





