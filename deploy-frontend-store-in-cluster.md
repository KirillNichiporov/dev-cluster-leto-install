# Деплой forntend-части

Перед тем как писать манифесты для фронта, необходимо подготовить бэк: точки входа ингресс, во первых настроить backend-протокол, чтобы при хотьбе с сервиса на сервис использовался https иначе не пропустит и вылетит 502 ошибка, так же настроить cors на точке входа, чтобы можно было осуществлять проверку(все параметры в манифесте ингресса)

```bash

nginx.ingress.kubernetes.io/enable-cors: "true"
nginx.ingress.kubernetes.io/cors-allow-methods: "PUT, GET, POST, OPTIONS"
nginx.ingress.kubernetes.io/cors-allow-origin: "*"
nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

```

Представленные настройки нужно внести в все ingress сервисов, которые будут использоваться(https во все, cors только в нужные)

Создаем манифесты для сервисов, вносим адреса сервисов бэка

### Деплой sss-store

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sss-store-deployment
  labels:
    app: sss-store
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sss-store
  template:
    metadata:
      labels:
        app: sss-store

    spec:
      containers:
      - name: sss-store-image
        image: git2.uonmap.com:5555/smartsafeschool/frontend/sss_store/sss_store:test
        imagePullPolicy: Always
        env:

        - name: VITE_AUTH_URL
          value: "https://auth-api-dev.smartsafeschool.com"
        - name: VITE_STORE_URL
          value: "https://store-api-dev.smartsafeschool.com"
        - name: VITE_CATALOG_URL
          value: "https://product-api-dev.smartsafeschool.com"
        - name: VITE_MEDIA_URL
          value: "https://media-api-dev.smartsafeschool.com"


        ports:
        - containerPort: 80
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: sss-store-service
spec:
  selector:
    app: sss-store
  ports:
  - name: port1
    protocol: TCP
    port: 80


```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sss-store-ingress
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
                name: sss-store-service
                port:
                  number: 80
  tls:
  - hosts:
    - store-dev.smartsafeschool.com 
    secretName: tls-secret

```

После создания манифестов отправляем все в кластер (указываем нужный namespace)

```bash

kubectl apply -f sss-store-deployment.yaml -n store
kubectl apply -f sss-store-service.yaml -n store
kubectl apply -f sss-store-ingress.yaml -n store

```

### Деплой sss-account-manager

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sss-account-manager-deployment
  labels:
    app: sss-account-manager
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sss-account-manager
  template:
    metadata:
      labels:
        app: sss-account-manager

    spec:
      containers:
      - name: sss-account-manager-image
        image: git2.uonmap.com:5555/smartsafeschool/frontend/sss_store/sss_account_manager:test
        imagePullPolicy: Always
        env:

        - name: VITE_AUTH_URL
          value: "https://auth-api-dev.smartsafeschool.com"
        - name: VITE_STORE_URL
          value: "https://store-api-dev.smartsafeschool.com"
        - name: VITE_CATALOG_URL
          value: "https://product-api-dev.smartsafeschool.com"
        - name: VITE_MEDIA_URL
          value: "https://media-api-dev.smartsafeschool.com"


        ports:
        - containerPort: 80
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: sss-account-manager-service
spec:
  selector:
    app: sss-account-manager
  ports:
  - name: port1
    protocol: TCP
    port: 80

```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sss-account-manager-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: manager-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sss-account-manager-service
                port:
                  number: 80
  tls:
  - hosts:
    - manager-dev.smartsafeschool.com 
    secretName: tls-secret

```

После создания манифестов отправляем все в кластер (указываем нужный namespace)

```bash

kubectl apply -f sss-account-manager-deployment.yaml -n store
kubectl apply -f sss-account-manager-service.yaml -n store
kubectl apply -f sss-account-manager-ingress.yaml -n store

```

### Деплой sss-account-vendor

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: sss-account-vendor-deployment
  labels:
    app: sss-account-vendor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sss-account-vendor
  template:
    metadata:
      labels:
        app: sss-account-vendor

    spec:
      containers:
      - name: sss-account-vendor-image
        image: git2.uonmap.com:5555/smartsafeschool/frontend/sss_store/sss_account_vendor:test
        imagePullPolicy: Always
        env:

        - name: VITE_AUTH_URL
          value: "https://auth-api-dev.smartsafeschool.com"
        - name: VITE_STORE_URL
          value: "https://store-api-dev.smartsafeschool.com"
        - name: VITE_CATALOG_URL
          value: "https://product-api-dev.smartsafeschool.com"
        - name: VITE_MEDIA_URL
          value: "https://media-api-dev.smartsafeschool.com"



        ports:
        - containerPort: 80
          protocol: TCP

      imagePullSecrets:
      - name: registry-cred

```

###### Service

```bash

apiVersion: v1
kind: Service
metadata:
  name: sss-account-vendor-service
spec:
  selector:
    app: sss-account-vendor
  ports:
  - name: port1
    protocol: TCP
    port: 80

```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sss-account-vendor-ingress
  namespace: store
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: b2b-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sss-account-vendor-service
                port:
                  number: 80
  tls:
  - hosts:
    - b2b-dev.smartsafeschool.com 
    secretName: tls-secret

```

После создания манифестов отправляем все в кластер (указываем нужный namespace)

```bash

kubectl apply -f sss-account-vendor-deployment.yaml -n store
kubectl apply -f sss-account-vendor-service.yaml -n store
kubectl apply -f sss-account-vendor-ingress.yaml -n store

```

Проверяем поднятые поды

```bash

kubectl get pods -n store

NAME                                                READY   STATUS    RESTARTS      AGE
auth-server-deployment-7c7686f5f9-pdrzw             1/1     Running   0             20h
mail-server-deployment-75cc5f9669-d8hrv             1/1     Running   0             20h
media-server-deployment-6d4857b49-fd4l9             1/1     Running   4 (20h ago)   20h
product-catalog-server-deployment-6cfdf7df4-6j8nj   1/1     Running   0             20h
sss-account-manager-deployment-577b47498d-j9vxv     1/1     Running   0             18h
sss-account-vendor-deployment-7d7fc97c67-mqmc7      1/1     Running   0             18h
sss-store-deployment-658b6774f8-jf682               1/1     Running   0             18h
store-server-deployment-fc47c9c9b-66rrm             1/1     Running   0             18h
warehouse-server-deployment-6cdf77698d-b5cjs        1/1     Running   0             20h

```
Отлично у подов статус running, на этом деплой закончен







