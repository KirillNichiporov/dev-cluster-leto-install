# Установка мониторинга

### Grafana and Prometheus

```bash

Установим Grafana из манифеста, но для начала создадим пространство имен

kubectl create namespace monitoring

Создадим секрет с сертификатом для TLS

kubectl create secret -n monitiring tls tls-secret --cert=/path/fullchain.pem --key=/path/privkey.pem

После применим манифест (представлен ниже)

kubectl apply -f grafana.yaml

```

###### Grafana 

```bash

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
               "access":"proxy",
                "editable": true,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus-service:8080",
                "version": 1
            }
        ]
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - name: grafana
          containerPort: 3000
        resources:
          limits:
            memory: "1Gi"
            cpu: "500m"
          requests:
            memory: "1Gi"
            cpu: "500m"
        volumeMounts:
          - mountPath: /var/lib/grafana
            name: grafana-storage
          - mountPath: /etc/grafana/provisioning/datasources
            name: grafana-datasources
            readOnly: false
      volumes:
        - name: grafana-storage
          emptyDir: {}
        - name: grafana-datasources
          configMap:
              defaultMode: 420
              name: grafana-datasources
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '3000'
spec:
  selector:
    app: grafana
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-grafana
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: grafana-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000

  tls:
  - hosts:
    - grafana-dev.smartsafeschool.com 
    secretName: tls-secret
```

```bash

Установим Prometheus из манифеста (представлен ниже), в то же пространство имен, что и Grafana

kubectl apply -f prometheus.yaml

```

###### Prometheus

```bash

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: devopscube demo alert
      rules:
      - alert: High Pod Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager:9093"
    scrape_configs:
      - job_name: kubernetes-nodes-cadvisor
        scrape_interval: 10s
        scrape_timeout: 10s
        scheme: https  # remove if you want to scrape metrics on insecure port
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          # Only for Kubernetes ^1.7.3.
          # See: https://github.com/prometheus/prometheus/issues/2916
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
        metric_relabel_configs:
          - action: replace
            source_labels: [id]
            regex: '^/machine\.slice/machine-rkt\\x2d([^\\]+)\\.+/([^/]+)\.service$'
            target_label: rkt_container_name
            replacement: '${2}-${1}'
          - action: replace
            source_labels: [id]
            regex: '^/system\.slice/(.+)\.service$'
            target_label: systemd_service_name
            replacement: '${1}'
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_endpoints_name]
          regex: 'node-exporter'
          action: keep
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics:8080']
      - job_name: 'kubernetes-cadvisor'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      
      - job_name: 'kubernetes-service-endpoints'
        kubernetes_sd_configs:
        - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
        - name: prometheus-storage-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus-server
    #type: NodePort
  ports:
    - port: 8080
      targetPort: 9090 
      # nodePort: 30003
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-prometheus
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: prometheus-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-service
                port:
                  number: 8080
  tls:
  - hosts:
    - prometheus-dev.smartsafeschool.com 
    secretName: tls-secret

```


Проверим поднятые поды

```bash

kubectl get pods -n monitoring
NAME                                     READY   STATUS    RESTARTS      AGE
grafana-54ccb64874-8ngqb                 1/1     Running   9 (9h ago)    43d
kube-state-metrics-84b67f9f48-ffwv7      1/1     Running   21 (9h ago)   78d
prometheus-deployment-8574d8cc9d-w4znx   1/1     Running   2 (9h ago)    10d

```

Ингресс имена надо добавить на прокси-машину и на dns локальный

# Установка системы сбора логов (Grafana Loki)

Создадим пространство имен

```bash

kubectl create namespace logging

```

Начнем установку Loki

```bash

установим helm репозиторий

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

Установим Loki

helm install loki grafana/loki-stack -n logging

```

Проверим поднятые поды 

```bash

kubectl get pods -n logging
NAME                  READY   STATUS    RESTARTS     AGE
loki-0                1/1     Running   2 (9h ago)   43h
loki-promtail-ftrkk   1/1     Running   2 (9h ago)   43h
loki-promtail-jt85w   1/1     Running   2 (9h ago)   43h
loki-promtail-vhmhg   1/1     Running   2 (9h ago)   43h
loki-promtail-x6xwx   1/1     Running   2 (9h ago)   43h

```

Установка закончена, сервис Loki интегрируется с Grafana, Promtail собирает логи и отправляет в Loki


# Установка Prometheus и node-exporter для сбора метрик на ВМ кластера

### Установка node-exporter (выполняется на всех ВМ кластера)

Качаем архив, распаковывем его, копируем бинарник и удаляем архив

```bash

wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz

tar xvf node_exporter-1.3.1.linux-amd64.tar.gz

cd node_exporter-1.3.1.linux-amd64

sudo cp node_exporter /usr/local/bin

cd ..

rm -rf ./node_exporter-1.3.1.linux-amd64

```

Создаем пользователя и назначаем его владельцем бинарника

```bash

sudo useradd --no-create-home --shell /bin/false node_exporter

sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

```

Создаем сервис node-exporter

```bash

sudo nano /etc/systemd/system/node_exporter.service

Добавляем в файл 

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

```

Стартуем сервис и включаем автозапуск

```bash

sudo systemctl daemon-reload

sudo systemctl start node_exporter

sudo systemctl enable node_exporter

```

порты открывать нет необходимости, ибо на ВМ кластера все открыты

Проверить работу можно с помощью 

```bash

curl http://ip_vm:91000/metrics

```

### Установка Prometheus (устанавливается на все ВМ кластера)

Создаем группу и каталоги для конфигураций

```bash

sudo groupadd --system prometheus

sudo useradd -s /sbin/nologin --system -g prometheus prometheus

sudo mkdir /var/lib/prometheus

for i in rules rules.d files_sd; do sudo mkdir -p /etc/prometheus/${i}; done

```

Загружаем файлы Prometheus

```bash

mkdir -p /tmp/prometheus && cd /tmp/prometheus

curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep browser_download_url | grep linux-amd64 | cut -d '"' -f 4 | wget -qi -

tar xvf prometheus*.tar.gz

cd prometheus*/

sudo mv prometheus promtool /usr/local/bin/

sudo mv prometheus.yml /etc/prometheus/prometheus.yml

sudo mv consoles/ console_libraries/ /etc/prometheus/

```
По пути /etc/prometheus/prometheus.yml можно настроить конфиг, но нам подходит дефолтный

Создаем сервис и даем права

```bash

sudo tee /etc/systemd/system/prometheus.service<<EOF
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP \$MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF


for i in rules rules.d files_sd; do sudo chown -R prometheus:prometheus /etc/prometheus/${i}; done

for i in rules rules.d files_sd; do sudo chmod -R 775 /etc/prometheus/${i}; done

sudo chown -R prometheus:prometheus /var/lib/prometheus/

```

Стартуем сервис и задаем автостарт

```bash

sudo systemctl daemon-reload

sudo systemctl start prometheus

sudo systemctl enable prometheus

```
Проверить сервис можно через systemctl status, порты открывать не нужно, он и так открыты (prometheus 9090)

На этом установка закончена остается дело за интеграцией Prometheus и Loki c Grafana

# Интеграция Loki и Prometheus c Grafana

### Prometheus

Для интеграции надо перейти в браузере по адресу grafana-dev.smartsafeschool.com, залогиниться с кредами (admin/k17n02i01)

На боковой панели выбрать connections затем datasource, после добавить новое для Prometheus 

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/25466edc-389a-4619-b9e7-a25a91580de5)

Далее указываем имя сервиса и порт

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/e262c55a-0ea5-4aa6-b6bb-c7a1aa50c729)

После тестируем коннект

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/fbd04e00-1cfc-40c1-a468-af81717ead42)
 
 
 Готово, подключение для нод аналогичное, только надо использовать из ip для адреса
 
 ![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/4f824d1c-c628-49cc-b7b4-8458c33a0ae8)


### Loki

Добавление Loki аналогично Prometheus, указываем имя сервиса и пространство имен с портом, а затем тестируем 

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/d0f54721-6d2c-4a7b-9a07-2b344c9b8dcb)



# Добавление Dashboards

### Service monitoring

Переходим во вкладку на боковой панели dashboards и жмем создать новое 

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/d2202669-43f1-429a-9e89-077347a0b130)

Выбираем dashboard или folder, чтобы потом туда поместить dashboard


Добавляем визуализацию

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/e3e4408a-25df-4099-aad5-d1d8abe2cf6b)

После выбираем datasource prometheus(для нод выбираем их подключения) нужные метрики и объект, жмем run queries, для которого собираются, потом жмем apply и сохраняем dashboard

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/ff37b204-c116-4ae5-8e09-4eb22cf2bc88)

### Service logging

Для логгинга добавляем dashboard и подключаем datasource Loki, выбираем контейнер для сбора логов, жмем run queries, потом apply и сохраняем

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/773e72f0-86dd-4252-80ca-f8c9d720906c)


Данные могут не появиться из-за того, что их нет, тогда можно просто увеличить время для поиска

![image](https://github.com/KirillNichiporov/dev-cluster-leto-install/assets/110092772/f4479344-f104-4c99-85fc-10381ee83757)

На этом по dashboards все, это простые настройки, дабы сделать прям серьезные, необходимо покопаться в конфигурациях

# Установка Kubernetes Dashboard

Для начала создаем пространство имен и секрет для tls

```bash

kubectl create namespace kubernetes-dashboard

kubectl create secret -n kubernetes-dashboard tls tls-secret --cert=/path/fullchain.pem --key=/path/privkey.pem

```

Затем устанавливаем kubernetes-dashboard 

```bash

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml -n kubernetes-dashboard

```

Проверим поды

```bash

kubectl get pods -n kubernetes-dashboard
NAME                                        READY   STATUS    RESTARTS      AGE
dashboard-metrics-scraper-7bc864c59-5m8j7   1/1     Running   2 (13h ago)   2d20h
kubernetes-dashboard-6c7ccbcf87-rkrtn       1/1     Running   2 (13h ago)   2d20h

```

Ставим ingress (приведен ниже)

```bash

kubectl apply -f kubernetes-dashboard-ingress.yaml -n kubernetes-dashboard


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
    - host: kubernetes-dashboard-dev.smartsafeschool.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 443
  tls:
  - hosts:
    - kubernetes-dashboard-dev.smartsafeschool.com 
    secretName: tls-secret


```

Содаем роли для токена входа

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  
 ``` 
  
```bash
 
 apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard

```

```bash

kubectl apply -f admin.yaml
kubectl apply -f role.yaml

```
Готово, после этого надо создавать токен доступа(kubernetes ради безопасности его чистит через время, поэтому создавать надо через время)

```bash

kubectl -n kubernetes-dashboard create token admin

```
ingress добавляем в прокси-машину и в dns локальный

При входе в браузере указываем токен, кторый создали

На этом с установками все
