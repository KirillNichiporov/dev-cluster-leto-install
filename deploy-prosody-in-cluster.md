# Подготовка к развертыванию prosody.

В первую очередь создадим namespace (делаем на машине администратора) 

```bash

kubectl create namespace prosody

```

Переходим на машину nfs, создаем на ней каталоги для хранения

```bash

mkdir /var/nfs/prosody

пользователя anon делаем владельцем, дабы кластер мог подключиться потом к каталогам

chown anon /var/nfs/prosody

```

Открываем файл exports, прописываем строки внутри и обновляем

```bash

vi /etc/exports

/var/nfs/prosody 192.168.1.0/24(rw,sync,all_squash,root_squash,anonuid=1001,anongid=1001)

exportfs -a

```

C nfs все, теперь установим provisioner в кластер

Заходим на машину администратора и выполняем команды, внутри команд нужно указать хост, а так же путь к каталогу, что будет использоваться и имя storageclass

```bash

helm install nfs-prosody-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.1.80 --set nfs.path=/var/nfs/prosody --set storageClass.name=prosody-storage

```

после выполнения проверим результат

```bash

kubectl get pods -n default

NAME                                                              READY   STATUS    RESTARTS        AGE
nfs-kafka-subdir-external-provisioner-nfs-subdir-external-wt72v   1/1     Running   2 (4d12h ago)   4d18h
nfs-monolith-subdir-external-provisioner-nfs-subdir-extern2prvm   1/1     Running   3 (4d12h ago)   11d
nfs-prosody-subdir-external-provisioner-nfs-subdir-externa8j7tq   1/1     Running   0               21h
nfs-redis-subdir-external-provisioner-nfs-subdir-external-4666f   1/1     Running   2 (4d12h ago)   4d18h


```

Далее создадим pvc для привязки к nfs 

###### prosody-pvc-config

```bash

apiVersion: v1
metadata:
  name: prosody-pvc-config
  annotations:
    volume.beta.kubernetes.io/storage-class: "prosody-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
      
```

      
###### prosody-pvc-plugins-custom

```bash

apiVersion: v1
metadata:
  name: prosody-pvc-plugins-custom
  annotations:
    volume.beta.kubernetes.io/storage-class: "prosody-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

      
```

Отправим конфигурацию в кластер в namespace prosody

```bash

kubectl apply -f prosody-pvc-config.yaml -n prosody

kubectl apply -f prosody-pvc-plugins-custom.yaml -n prosody

```

У pvc должен быть статус Bound

```bash

kubectl get pvc -n prosody

NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
prosody-pvc-config           Bound    pvc-41003667-117b-4e1c-9d44-458cbc4c533a   5Gi        RWX            prosody-storage   18h
prosody-pvc-plugins-custom   Bound    pvc-38e5e4f5-65ad-4b81-b931-05e452a9b37d   10Gi       RWX            prosody-storage   18h

```


### Создание секрета с кредами для доступа к gitlab registry

На машине администратора выполняем команду (секрет надо поместить в namespace monolith иначе приложение его не найдет)

```bash

kubectl create secret -n prosody docker-registry registry-cred --docker-server=https://git2.uonmap.com:5555 --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>

```
### Создание секрета TLS

На машине администратора выполняем команду (секрет надо поместить в namespace monolith иначе приложение его не найдет)

```bash

kubectl create secret -n prosody tls tls-secret --cert=/path/fullchain.pem --key=/path/privkey.pem

```

# Развертка prosody

Для развертки подготовлен манифест Deployment, внутри указаны основные настройки для приложения и конкретно контейнера, локальные папки поделючены к nfs через pvc, указаны labels, по которым подключится сервис к контенеру и обеспечит доступ, переданы креды к приватному репозиторию имеджей registry-cred, можно регулировать количество реплик, прописаны используемые порты. 

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prosody-deployment
  labels:
    app: prosody
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prosody
  template:
    metadata:
      labels:
        app: prosody

    spec:
      containers:
      - name: prosody-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/smartdevice/docker_jitsi_meet:prosody_latest
        env:

        - name: TZ
          value: "UTC"
        - name: API_XMPP_USER
          value: "api.server"
        - name: API_XMPP_PASSWORD
          value: "1cf44b4e337f54270f7b870a90415b8e"
        - name: AUTH_TYPE
          value: "jwt"
        - name: ENABLE_AUTH
          value: "1"

        - name: GLOBAL_MODULES
          value: "blocklist"
        - name: JICOFO_AUTH_USER
          value: "focus"
        - name: JICOFO_AUTH_PASSWORD
          value: "5d5c6daee9e0ad63414f4973b9bf9f80"
        - name: JVB_AUTH_USER
          value: "jvb"
        - name: JVB_AUTH_PASSWORD
          value: "b9a48c1b592a7ecefe5fb35c943e702e"

        - name: JWT_APP_ID
          value: "smart_safe_school_app"
        - name: JWT_APP_SECRET
          value: "3752143452fc0f705564ef4086f3aab6"
        - name: PUBLIC_URL
          value: "https://messenger-jitsi-dev.smartsafeschool.com"
        - name: XMPP_AUTH_DOMAIN
          value: "auth.ubuntu1.smartsafeschool.com"
        - name: XMPP_GUEST_DOMAIN
          value: "guest.ubuntu1.smartsafeschool.com"

        - name: XMPP_MUC_DOMAIN
          value: "muc.ubuntu1.smartsafeschool.com"
        - name: XMPP_INTERNAL_MUC_DOMAIN
          value: "internal-muc.ubuntu1.smartsafeschool.com"
        - name: XMPP_RECORDER_DOMAIN
          value: "recorder.ubuntu1.smartsafeschool.com"
        - name: XMPP_SERVER
          value: "xmpp.ubuntu1.smartsafeschool.com"

        ports:
        - containerPort: 5222
          protocol: TCP
        - containerPort: 5347
          protocol: TCP
        - containerPort: 5280
          protocol: TCP
        volumeMounts:
        - name: prosody-config
          mountPath: /config:Z
        - name: prosody-plugins-custom
          mountPath: /prosody-plugins-custom:Z

      volumes:
      - name: prosody-config
        persistentVolumeClaim:
          claimName: prosody-pvc-config
      - name: prosody-plugins-custom
        persistentVolumeClaim:
          claimName: prosody-pvc-plugins-custom

      imagePullSecrets:
      - name: registry-cred

```

Теперь выполняем команду для деплоя prosody (запускаем в нужном namespace)

```bash

kubectl apply -f prosody-deployment.yaml -n prosody

статус должен быть running

kubectl get pods -n prosody

NAME                                  READY   STATUS    RESTARTS   AGE
prosody-deployment-5c4dcb87bb-vnkhs   1/1     Running   0          17h


```


Далее необходимо настроить сервис и ингресс для подключения к prosody

В сервисе указываем selector как в deployment labels, чтобы иметь доступ к нужному контейнеру и порты, к котрым необходим доступ у сервиса.

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: prosody-service
spec:
  selector:
    app: prosody 
  ports:
  - name: public-port
    protocol: TCP
    port: 5222
  - name: port-1
    protocol: TCP
    port: 5347
  - name: port-2
    protocol: TCP
    port: 5280

```

###### Ingress

В ингресс указываем dns имя сервиса и порт, по которому нужно идти на сервис, так же секрет для tls

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prosody-ingress
  namespace: prosody
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: messenger-jitsi-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prosody-service
                port:
                  number: 5222
  tls:
  - hosts:
    - messenger-jitsi-dev.smartsafeschool.com 
    secretName: tls-secret

```

Добавляем конфигурацию в кластер (не забываем про namespace)

```bash

kubectl apply -f prosody-service.yaml -n prosody

kubectl apply -f prosody-ingress.yaml -n prosody

```


На машине k8s-proxy в каталоге /etc/nginx/sites-enabled/ в файл ingress добавляем dns

```bash

Проверяем конфигурации 

nginx -t

nginx -s reload

```
Далее переконфигурируем манифест deployment для monolith, чтобы он подключался к prosody в кластере, все переменные практически те же, но надо поменять APP_XMPP_HOST=prosody-service.prosody здесь указывается сервис и namespace prosody в кластере,SPRING_PROFILES_ACTIVE=pruduction,xmpp

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
        - name: APP_PUBLIC_URL
          value: "http://mn-dev.smartsafeschool.com"
        - name: SPRING_PROFILES_ACTIVE
          value: "production,xmpp"
        - name: DB_HOST
          value: "ext-mariadb"
        - name: DB_USER
          value: "smart_devices"
        - name: DB_PASSWORD
          value: "snr!453)FvmzP"
        - name: DB_SCHEME
          value: "smartdevices"
        - name: DB_PARAMETERS
          value: "createDatabaseIfNotExist=true&useUnicode=true&characterEncoding=UTF-8&allowPublicKeyRetrieval=true&serverTimezone=UTC&enabledTLSProtocols=TLSv1.2&useSSL=true"

        - name: MAIL_HOST
          value: "mail.pm.uonmap.com"
        - name: MAIL_PORT
          value: "587"
        - name: MAIL_PASSWORD
          value: "uEA6GK47GZ"
        - name: MAIL_USERNAME
          value: "smartsave@pm.uonmap.com"
        - name: MAIL_FROM
          value: "no-reply@smartsafeschool.com"

        - name: APNS_DEVELOP_SERVER
          value: "false"
        - name: APNS_P12_PASSWORD
          value: "WfYvoRv5QTJOqOW23kD2"
        - name: NETTY_MODULE_TOKEN
          value: "954e1ece-494b-48fd-86c2-58b017795b8a"
        - name: APP_CORS_ALLOWED_ORIGINS
          value: "https://admin.mn-dev.smartsafeschool.com"


        - name: APP_XMPP_HOST
          value: "prosody-service.prosody"
        - name: APP_XMPP_XMPP_DOMAIN
          value: "ubuntu1.smartsafeschool.com"
        - name: APP_XMPP_USER
          value: "api.server"
        - name: APP_XMPP_PASSWORD
          value: "1cf44b4e337f54270f7b870a90415b8e"
        - name: APP_XMPP_TRUST_ALL_SSL
          value: "true"

        - name: APP_JITSI_XMPP_DOMAIN
          value: "ubuntu1.smartsafeschool.com"
        - name: APP_JITSI_JWT_APP_SECRET
          value: "3752143452fc0f705564ef4086f3aab6"
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

Oбновляем monolith (файлы в каталогк монолита)

kubectl apply -f monolith-deployment.yaml -n monolith


Проверяем статус и ответ от сервиса monolith

```bash

kubectl get pods -n monolith

NAME                                   READY   STATUS    RESTARTS      AGE
monolith-deployment-677754855b-qwsmr   1/1     Running   7 (17h ago)   17h

curl https://mn-dev.smartsafeschool.com/api/admin/version/

{"status":401,"message":null,"error":"Full authentication is required to access this resource","timestamp":"2023-04-12T07:15:13.307Z"}

```

Развертка на этом закончена







