# Общая настройка виртуальных машин

В начале необходимо настроить 5 виртуальных машин (ВМ):
- k8s-admin — ВМ для администратора, чтобы управлять кластером вручную, с нее и будет разворачиваться весь кластер, а также установлены утилиты для управления (helm и kubectl).
- k8s-master-1 — мастер нода для управления кластером (внутренние процессы и т.д.)
- k8s-ingress-1 — ингресс контроллер
- k8s-worker-1 — первая нода для разворачивания сервисов
- k8s-worker-2 — вторая нода для разворачивания сервисов
На каждой из машин при установке нужно установить статический ip, в качестве DNS-серверов были поставлены 8.8.8.8 и 8.8.4.4 ( можно поставить свои, но главное, чтоб не перекрывался доступ ко всем необходимым ресурсам для кластера), удалить swap, отключить selinux и firewall.

### Выполнить команды по установке (на всех машинах):

```bash

sudo apt-get install wget curl git screen python3-pip sshpass

```

### для Docker:

```bash

sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release


sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```

# Настройка машины администратора

Далее переходим на машину администратора (k8s-admin), здесь необходимо установить ansible для удаленного управления машинами кластера при устнаовке кластера с помощью kubespray.

### Установка Ansible

Так как до этого был установлен python3-pip, то с его помощью устанавливаем ansible:

```bash

pip3 install ansible

```

### Проброс SSH-ключей между машинами

Генерируем ключи на каждой из машин с помощью:

```bash

ssh-keygen

```
passphrase не задаем

Далее с машин кластера на машину администратора и с нее на машины кластера с помощью:

```bash

ssh-copy-id user@vm-ip

```

### Установка Kubectl

```bash

curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

```

Проверка 

```bash

kubectl version

```

### Установка Helm как установщика прочих инструментов

```bash

curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm

```

### Установка K9s для удобства управления

###### Установка Go

```bash

wget https://go.dev/dl/go1.19.linux-amd64.tar.gz

sudo rm -rf /usr/local/go && sudo  tar -C /usr/local -xzf go1.19.linux-amd64.tar.gz

ls -l /usr/local/go/bin/

export PATH=$PATH:/usr/local/go/bin/

go version

```

###### Установка K9s

```bash

wget https://github.com/derailed/k9s/releases/download/v0.26.3/k9s_Linux_x86_64.tar.gz

sudo tar -C /usr/local/bin -xzf k9s_Linux_x86_64.tar.gz

```

# Установка кластера

На машине администратора клонируем репозиторий kubespray:

```bash

git clone https://github.com/kubernetes-sigs/kubespray.git

```
Переходим в директорию, в ней будут осуществляться все основные нстройки и запуск плейбука для установки кластера

```bash

cd kubespray

```

После, делаем копию каталога sample внутри этой копии (dev) будет размещаться используемый для установки inventory файл, устанавливаем плагины из предоставленного файла( их несколько, выберите вам подходящий), после объявляем переменную состоящую из адресов ВМ для кластера (адреса перечисляем через пробел), создаем конфигурационный файл в каталоге, где должен храниться inventory файл, с помощью скрипта от kubespray.

```bash

cp -rfp inventory/sample inventory/dev

pip3 install -r requirements-*.txt

declare -a IPS=(vm_ip)

CONFIG_FILE=inventory/dev/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

```

Внутри inventory файла размещаем имена машин по группам, назначаем мастеров(etcd и control-plane), а такде указываем пользователя и файл с ключами к ВМ кластера(предварительно можно проверить работу модулем ping от ansible) 

```bash

all:
  vars:
    ansible_user: user
    ansible_public_ssh_key_path: /home/user/.ssh/authorized_keys
  hosts:
    k8s-master-1:
      ansible_host: 192.**.**.**
      ip: 192.**.**.**
      access_ip: 192.**.**.**
    k8s-worker-1:
      ansible_host: 192.**.**.**
      ip: 192.**.**.**
      access_ip: 192.**.**.**
    k8s-worker-2:
      ansible_host: 192.**.**.**
      ip: 192.**.**.**
      access_ip: 192.**.**.**
    k8s-ingress-1:
      ansible_host: 192.**.**.**
      ip: 192.**.**.**
      access_ip: 192.**.**.**
  children:
    kube_control_plane:
      hosts:
        k8s-master-1:
    kube_node:
      hosts:
        k8s-worker-1:
        k8s-worker-2:
        k8s-ingress-1:
    etcd:
      hosts:
        k8s-master-1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}

```

Если все до этого настроили правильно и ничего не пропустили, то запускаем плейбук от kubespray, если в качестве пользователей на ВМ используются не root, то необходимо передать ansible привилегии sudo, вот полная команда для установки:

```bash

ansible-playbook -i inventory/dev/hosts.yaml -b cluster.yml --ask-pass-become

```

После отработки плейбука переходим на машину k8s-master-1 проверяем статус нод(если плейбук отработал без ошибок), у всех должен быть статус Ready:

```bash

kubectl get nodes

```

Также настроим ingress ноду, ставим метку ingress и прописываем правила запуска подов на ноде, в случае с ingress нодой на ней будет запускаться только контроллер(в манифесте прописаны правила и поставлены метки для запуска подов контроллера)

```bash

kubectl label node k8s-ingress-1 node-role.kubernetes.io/ingress=ingress

kubectl taint nodes k8s-ingress-1 key1=value1:NoSchedule

```

Для того, чтобы управлять с машины администратора кластером нужно добавить файл config из /root/.kube/ на мастер-машине в предварительно созданную папку .kube внутри домашнего каталога пользователя, от имени которого будет осуществляться управление, на машине администратора и отредактировать адрес сервера в файле config вместо 127.0.0.1 на адрес мастер-машины.

Проверяем на машине администратора:

```bash

kubectl get nodes

```
Результат команды аналогичный результату на мастер-машине


















