# Деплой forntend-части

Создаем манифест для сервисa, вносим адреса сервиса бэка

###### Deployment

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: monolith-web-deployment
  labels:
    app: monolith-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monolith-web
  template:
    metadata:
      labels:
        app: monolith-web

    spec:
      containers:
      - name: monolith-web-image
        image: git2.uonmap.com:5555/smartsafeschool/frontend/sss_web_account/web_account:test
        imagePullPolicy: Always
        env:

        - name: VITE_API_URL
          value: "http://monolith-service.monolith:8081/api/"
        - name: VITE_I18N_LOCALE
          value: "en"
        - name: VITE_I18N_FALLBACK_LOCALE
          value: "en"
        - name: VITE_SYSTEM_ROLES
          value: '{"super_admin": "ROLE_SUPER_ADMIN","support": "ROLE_SUPPORT","admin": "ROLE_ADMIN","authority": "ROLE_AUTHORITY"}'

        - name: VITE_NON_SYSTEM_ROLES
          value: '{"student": "ROLE_STUDENT", "parent": "ROLE_PARENT", "staff": "ROLE_STAFF"}'
        - name: VITE_PROTECTED_ROLES
          value: '["ROLE_SUPER_ADMIN", "ROLE_ADMIN", "ROLE_STAFF", "ROLE_PARENT", "ROLE_STUDENT", "ROLE_SUPPORT", "ROLE_AUTHORITY"]'
        - name: VITE_PROTECTED_TAGS
          value: '{"ADMINISTRATOR": "Administrators","STAFF": "Staff","PARENT": "Parents","STUDENT": "Students"}'
        - name: VITE_DEFAULT_ROLE
          value: "ROLE_STUDENT"
        - name: VITE_MODULES
          value: '["school", "users", "news", "notifications", "checklists", "buses", "smarthubs", "externalSensors", "things", "emergencies", "moodle"]'


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
  name: monolith-web-service
spec:
  selector:
    app: monolith-web 
  ports:
  - name: http
    protocol: TCP
    port: 80

```


###### Ingress

```bash

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monolith-web-ingress
  namespace: monolith
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: admin-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monolith-web-service
                port:
                  number: 80
  tls:
  - hosts:
    - admin-dev.smartsafeschool.com 
    secretName: tls-secret

```

После создания манифестов отправляем все в кластер (указываем нужный namespace)

```bash

kubectl apply -f monolith-web-deployment.yaml -n monolith
kubectl apply -f monolith-web-service.yaml -n monolith
kubectl apply -f monolith-web-ingress.yaml -n monolith

```


Проверяем поднятый под

```bash

kubectl get pods -n monolith

NAME                                       READY   STATUS    RESTARTS     AGE
monolith-deployment-77f8c674bd-sqg2z       1/1     Running   7 (8h ago)   34d
monolith-web-deployment-56844f8dc6-xjgsp   1/1     Running   0            84m


```
Отлично у пода статус running, на этом деплой закончен

Остается добавить запись admin-dev.smartsafeschool.com IN A 192.168.1.79 на DNS и выполнить команду

```bash

systemctl named reload

```
Также добавляем запись в proxy и выполняем


```bash

nginx -t

nginx -s reload

```



