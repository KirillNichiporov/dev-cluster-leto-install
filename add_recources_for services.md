# Добавление лимитов для сервиса

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
        image: git2.uonmap.com:5555/smartsafeschool/backend/sss_services/sss_services/3s-mail/mail-service:dev-latest
        imagePullPolicy: Always
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m" 
          limits:
            memory: "400Mi"
            cpu: "700m"
        env:
        - name: spring.profiles.active
          value: "k8s-dev"
        - name: config.security.h1.white-headers
          value: "all"
        - name: spring.mail.username
          value: "dev-service@smartsafeschool.com"
        - name: spring.mail.from
          value: "dev-service@smartsafeschool.com"
        - name: spring.mail.password
          value: "DemoTest-2023"
        - name: spring.kafka.bootstrap-servers
          value: "kafka.kafka:9092"

        ports:
        - containerPort: 2443
          protocol: TCP
      imagePullSecrets:
      - name: registry-cred

```

В деплойменте необходимо добавить поле resources с настройками ЦПУ и Оперативной памяти, но важно учитывать общую сумму ресурсов по памяти и ядрам,ибо заявленное кол-во бронируется и планировщи кластера не сможет поднять под, если не будет свободного ресурса, это делается для того,чтобы при нагрузке ресурсов не потреблялось математически больше,чем вообще существует,
в нашем случае для ЦПУ можно не указывать, так как упор на память и потребление ЦПУ существенно только при старте контейнеров.

После обновляем деплоймент 

```bash

kubectl apply -f <DEPLOYMENT>.yaml -n <NAMESPACE>

```

# Добавление лимитов на Kafka и Redis

Предварительно очистим релиз Helm и удалим все, чтоб поставить новое

Посмотреть релизы Helm

```bash

helm list -n <NAMESPACE>

```


Далее удаляем релизы

```bash

helm unnstall <RELEASE NAME> -n <NAMESPACE>

```

Далее для Kafka и Redis лежат в их каталогах файлы values.yaml(хранятся настройки,можно посмотреть, там назначены хранилища и ресурсы, на Kafka отключен sasl)

Выполняем с помощью них команду

```bash

helm install <RELEASE NAME> -f values.yaml -n <NAMESPACE> bitnami/(kafka или redis)

```

НЕ ЗАБЫТЬ ПЕРЕЗАПУСТИТЬ UI, а также внимательно подставлять все переменные и проверить через UI коннекты















