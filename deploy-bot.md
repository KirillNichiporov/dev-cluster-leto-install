# Подготовка БД

На машине 192.168.1.71 выполняем команды по настройке пользователя и новой бд

```bash

su - postgres

psql

create user dev_bot_admin with password 'your_pass';

create database dev_bot_db with owner dev_bot_admin;

```

В файле /etc/postgresql/14/main/pg_hba.conf прописываем строку 

```bash

host	all		dev_bot_admin		 192.168.1.0/24	md5

```

После выполняем команду 

```bash

systemctl reload postgresql

```

БД настроена

Теперь загрузим манифесты в кластер

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: bot-deployment
  labels:
    app: bot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bot
  template:
    metadata:
      labels:
        app: bot

    spec:
      containers:
      - name: bot-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/smartdevice/openai_server:dev-latest
        imagePullPolicy: Always
        env:

        - name: CONFIG_AI_REQUEST_HISTORY_LIFETIME_DAYS
          value: "0"
        - name: DB_PASSWORD
          value: "PrM48103DsqyU12"
        - name: DB_URL
          value: "db-postgre-dev.smartsafeschool.com:5432/dev_bot_db"
        - name: DB_USER_NAME
          value: "dev_bot_admin"
        - name: OPEN_AI_TOKENS
          value: "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,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"
        - name: RESTFUL_MONOLITH_SERVER_HOST
          value: "http://monolith-service"




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
  name: bot-ingress
  namespace: monolith
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: dev-bot-api.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bot-service
                port:
                  number: 8080
  tls:
  - hosts:
    - dev-bot-api.smartsafeschool.com 
    secretName: tls-secret


```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: bot-service
spec:
  selector:
    app: bot
  ports:
  - name: port1
    protocol: TCP
    port: 8080


```

Выполняем команды 

```bash

kubectl apply -f bot-service.yaml -n monolith
kubectl apply -f bot-ingress.yaml -n monolith
kubectl apply -f bot-deployment.yaml -n monolith

```
Проверим под

```bash

NAME                                       READY   STATUS    RESTARTS       AGE
bot-deployment-85f74d9bc6-lqnzp            1/1     Running   0              17m
monolith-deployment-8477fbfc7d-tkv7m       1/1     Running   9 (131m ago)   25d
monolith-web-deployment-5d7655cc86-27hcr   1/1     Running   1 (131m ago)   47h

```
Статус Running, отлично

Теперь можно проверить

```bash

curl https://dev-bot-api.smartsafeschool.com/api/version

{"message":"1.0 [14.08.2023 14:29:43]","status":200,"error":null,"timestamp":"2023-08-17T15:52:28.279+03"}

```

Готово















