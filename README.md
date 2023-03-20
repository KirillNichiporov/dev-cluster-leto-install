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

### Установка кластера

На машине администратора клонируем репозиторий kubespray:

```bash

git clone https://github.com/kubernetes-sigs/kubespray.git

```
Переходим в директорию, в ней будут осуществляться все основные нстройки и запуск плейбука для установки кластера

```bash

cd kubespray

```

После, делаем копию каталога sample внутри этой копии (dev) будет размещаться используемый для установки inventory файл, устанавливаем плагины из предоставленного файла( их несколько, выберите вам подходящий), после объявляем переменную состоящую из адресов ВМ для кластера (адреса перечисляем через пробел), создаем конфигурационный файл в каталоге, где должен храниться inventory файл, с помощью скрипта от kubespray.

cp -rfp inventory/sample inventory/dev

apt install -f requirements-*.txt

declare -a IPS=(vm_ip)

CONFIG_FILE=inventory/dev/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

all:
  vars:
    ansible_user: user
    ansible_public_ssh_key_path: /home/user/.ssh/authorized_keys
  hosts:
    k8s-master-1:
      ansible_host: 192.168.1.74
      ip: 192.168.1.74
      access_ip: 192.168.1.74
    k8s-worker-1:
      ansible_host: 192.168.1.75
      ip: 192.168.1.75
      access_ip: 192.168.1.75
    k8s-worker-2:
      ansible_host: 192.168.1.76
      ip: 192.168.1.76
      access_ip: 192.168.1.76
    k8s-ingress-1:
      ansible_host: 192.168.1.77
      ip: 192.168.1.77
      access_ip: 192.168.1.77
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










