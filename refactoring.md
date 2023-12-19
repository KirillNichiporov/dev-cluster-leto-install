# Файловая система

Структуру ФС привести к виду как у dev,test сохранив все старое отдельно и не удаляя лишнего, так же многое можно подсмотреть в файлах на dev,test

# Перемещение сервисов в новые namespaces

Для начала создаем все нужные неймспейсы

```bash

kubectl create namespace <new-namespace>

```

Так же создаем секреты(регистри и  для tls)

```bash

kubectl create secret -n moodle docker-registry registry-cred --docker-server=https://git2.uonmap.com:5555 --docker-username=user_service --docker-password=BELpzoxx_7vsYRpUKCWp --docker-email=kirill.nichiporov.01@mail.ru

kubectl create secret tls tls-secret --cert=/PATH/fullchain.pem --key=/PATH/privkey.pem -n <new-namespace>

```

### basic (auth, mail, media)

В неймспейсе store удаляем ингрессы, деплойменты и сервисы наших приложений(указаны в заголовке), можно удалять из k9s

```bash

kubectl delete deployment/ingress/service (выбрать нужное) <name-deployment/ingress/service> -n store

Пример:

kubectl delete deployment auth-server-deployment -n store

```

После удаления запускаем все в basic, но перед этим заходим в манифесты ингресса и УДАЛЯЕМ строчку namespace: store

```bash

kubectl apply -f file-manifest(манифест yaml для приложения deployment/ingress/service) -n basic

Пример:

kubectl apply -f auth-deployment.yaml -n basic

```

Проверяем сервисы, запущены ли они, так же можно проверить логи

```bash

kubectl get pods -n basic

```

Так же надо в файле monolith-deployment монолита отредактировать переменную APP_CONFIG_MEDIA_HOST и поставить http://media-server-service.basic:4443

после обновить

```bash

kubectl apply -f monolith-deployment.yaml -n monolith

```

### financial (pmt)

В неймспейсе store удаляем ингрессы, деплойменты и сервисы наших приложений(указаны в заголовке), можно удалять из k9s

```bash

kubectl delete deployment/ingress/service (выбрать нужное) <name-deployment/ingress/service> -n store

Пример:

kubectl delete deployment pmt-server-deployment -n store

```

После удаления запускаем все в financial, но перед этим заходим в манифесты ингресса и УДАЛЯЕМ строчку namespace: store

```bash

kubectl apply -f file-manifest(манифест yaml для приложения deployment/ingress/service) -n financial

Пример:

kubectl apply -f pmt-deployment.yaml -n financial

```

Проверяем сервисы, запущены ли они, так же можно проверить логи

```bash

kubectl get pods -n financial

```

### ai (bot)

В неймспейсе monolith удаляем ингрессы, деплойменты и сервисы наших приложений(указаны в заголовке), можно удалять из k9s

Так же надо в файле openai-server-deployment монолита отредактировать переменную RESTFUL_MONOLITH_SERVER_HOST и поставить http://monolith-service.monolith:8081

```bash

kubectl delete deployment/ingress/service (выбрать нужное) <name-deployment/ingress/service> -n monolith

Пример:

kubectl delete deployment bot-deployment -n monolith

```

В манифестах  bot необходимо слово bot заменить на openai-server(кроме домена), можно подсмотреть на dev,test

Запускаем все в ai, но перед этим заходим в манифесты ингресса и УДАЛЯЕМ строчку namespace: monolith

```bash

kubectl apply -f file-manifest(манифест yaml для приложения deployment/ingress/service) -n ai

Пример:

kubectl apply -f openai-server-deployment.yaml -n ai

```

Проверяем сервисы, запущены ли они, так же можно проверить логи

```bash

kubectl get pods -n ai

```

### lms (moodle)

В неймспейсе moodle удаляем ингрессы, деплойменты и сервисы наших приложений(указаны в заголовке), можно удалять из k9s

```bash

kubectl delete deployment/ingress/service/pvc (выбрать нужное) <name-deployment/ingress/service/pvc> -n moodle

Пример:

kubectl delete deployment moodle-deployment -n monolith

```

В манифестах  moodle необходимо слово moodle заменить на moodle-external(кроме домена), можно подсмотреть на dev,test

Запускаем все в lms, но перед этим заходим в манифесты ингресса и УДАЛЯЕМ строчку namespace: moodle

PVC запускаем первым и проверяем статус через kubectl get pvc -n lms, статус должен быть bound

```bash

kubectl apply -f file-manifest(манифест yaml для приложения deployment/ingress/service/pvc) -n lms

Пример:

kubectl apply -f moodle-external-deployment.yaml -n lms

```

Проверяем сервисы, запущены ли они, так же можно проверить логи

```bash

kubectl get pods -n lms

```

# GRPC для auth

Редактируем auth-deployment.yaml, открывая порт микросервису

```bash

        ports:
        - containerPort: 1443
          protocol: TCP
        - containerPort: 11443

```


Создаем service внутри файла auth-service.yaml, разделители обязательно, инче не подтянется, интересует второй сервис

```bash

Примерно такой вид:

---
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
---
apiVersion: v1
kind: Service
metadata:
  name: auth-server-grpc-service
spec:
  selector:
    app: auth-server
  ports:
  - name: port1
    protocol: TCP
    port: 11443


```

Затем создаем ингресс на тот же домен в файле auth-ingress.yaml, интересует второй ингресс, первый не трогаем

```bash

Примерно такой вид:

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-server-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;
spec:
  rules:
    - host: dev-auth-api.smartsafeschool.com
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
      - dev-auth-api.smartsafeschool.com 
      secretName: tls-secret

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-server-grpc-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  rules:
    - host: dev-auth-api.smartsafeschool.com
      http:
        paths:
          - path: /authServerClient.AuthServerClientService/
            pathType: Prefix
            backend:
              service:
                name: auth-server-grpc-service
                port:
                  number: 11443

          - path: /grpc.health.v1.Health/
            pathType: Prefix
            backend:
              service:
                name: auth-server-grpc-service
                port:
                  number: 11443

          - path: /grpc.reflection.v1alpha.ServerReflection/
            pathType: Prefix
            backend:
              service:
                name: auth-server-grpc-service
                port:
                  number: 11443

  tls:
    - hosts:
      - dev-auth-api.smartsafeschool.com 
      secretName: tls-secret


```

Далее обновляем файлы начиная с деплоймента


```bash

kubectl apply -f your-manifest.yaml -n basic

```

Далее необходимо настроить прокси, создать сниппеты( каталог snippets в каталоге nginx) как на тесте и деве, затем подключить(include) в конфигах (sites-enabled) удалив заголовки, что вынесены в сниппеты, что не вынесены оставить, так же удалить запись домена auth-api из конфига ingress, она будет в auth.smartsafeschool.com, все можно подсмотреть на дев и тест, адреса и именя выставлять в соответствии с прод 

Закончив редактирование проверяем конфигурацию и обновляем

```bash

nginx -t

nginx -s reload

```







