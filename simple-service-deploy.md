# Деплой simple-service
Для деплоя подготовлены манифесты, также манифесты сервисов,что будут работать через nodeport, создаем пространство имен и секрет с кредами для registry

```bash

kubectl create namespace simple-services

kubectl create secret -n simple-services docker-registry registry-cred --docker-server=https://git2.uonmap.com:5555 --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>

```


### Deployments

###### Simple1

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-service1-deployment
  labels:
    app: simple-service1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-service1
  template:
    metadata:
      labels:
        app: simple-service1

    spec:
      containers:
      - name: simple-service1-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/simple_services_jvm:serv1-dev
        imagePullPolicy: Always
        env:

        - name: spring.application.name
          value: "simple-service1"
        - name: server.port
          value: "8081"
        - name: server.ssl.enabled
          value: "false"
        - name: config.client-server-url
          value: "http://simple-service2.simple-services:8082"
        - name: config.client-server-point
          value: "/simple-service2/hello-world"
        - name: log4j.debug
          value: "true"

        ports:
        - containerPort: 8081
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred


```


###### Simple2


```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-service2-deployment
  labels:
    app: simple-service2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-service2
  template:
    metadata:
      labels:
        app: simple-service2

    spec:
      containers:
      - name: simple-service2-image
        image: git2.uonmap.com:5555/smartsafeschool/backend/simple_services_jvm:serv2-dev
        imagePullPolicy: Always
        env:

        - name: spring.application.name
          value: "simple-service2"
        - name: server.port
          value: "8082"
        - name: server.ssl.enabled
          value: "false"
        - name: log4j.debug
          value: "true"

        ports:
        - containerPort: 8082
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred


```

###### Simple3


```bash

apiVersion: v1
kind: Service
metadata:
  name: simple-service2
spec:
  selector:
    app: simple-service2
  type: NodePort
  ports:
  - name: port1
    protocol: TCP
    port: 8082
    nodePort: 32000

```

### Services

###### Simple1

```bash

apiVersion: v1
kind: Service
metadata:
  name: simple-service1
spec:
  selector:
    app: simple-service1
  type: NodePort
  ports:
  - name: port1
    protocol: TCP
    port: 8081
    nodePort: 31000

```

###### Simple2


```bash

apiVersion: v1
kind: Service
metadata:
  name: simple-service2
spec:
  selector:
    app: simple-service2
  type: NodePort
  ports:
  - name: port1
    protocol: TCP
    port: 8082
    nodePort: 32000


```


###### Simple3

```bash

apiVersion: v1
kind: Service
metadata:
  name: simple-service2
spec:
  selector:
    app: simple-service2
  type: NodePort
  ports:
  - name: port1
    protocol: TCP
    port: 8082
    nodePort: 32000

```

Выполняем команды(нужный нам неймспейс выбрать не забываем)

```bash

kubectl apply -f simple1-deployment.yaml -n simple-services
kubectl apply -f simple2-deployment.yaml -n simple-services
kubectl apply -f simple3-deployment.yaml -n simple-services

kubectl apply -f simple1-service.yaml -n simple-services
kubectl apply -f simple2-service.yaml -n simple-services
kubectl apply -f simple3-service.yaml -n simple-services


```

Проверим поды

```bash

kubectl get pods -n simple-services

NAME                                          READY   STATUS    RESTARTS   AGE
simple-service1-deployment-849f57dd65-w4hsw   1/1     Running   0          108m
simple-service2-deployment-55c8c76f8d-5g86r   1/1     Running   0          113m
simple-service3-deployment-5ddf5d6874-56t49   1/1     Running   0          111m

```

Поды поднялись, теперь дернем наши сервисы

```bash

curl http://192.168.1.77:30003/simple-service3/hello-world
Hello world from [Simple Service 3]!

теперь дернем второй сервис используя первый

curl http://192.168.1.77:31000/simple-service1/request
Hello world from [Simple Service 2]!

```
Отлично, сервисы видят друг друга 

При включенном ssl обращение сервисов внутри кластера невозможно из-за сертификата он сделан на домен .smartsafeschool.com , а обращается сервси 1 на сервис 2 по внутрикластерному адресу https://simple2-service.simple-services:8082, можно решить вопрос сделав сертификат на все домены(если надо ssl в этом случае он не нужен)

Если включить ssl, то можно дергать сервис по имени ингресс машины
```bash

curl https://k8s-ingress-1.smartsafeschool.com:31000/simple-service1/hello
Hello from [Simple Service 1]!

```
