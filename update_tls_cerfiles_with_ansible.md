# Подгототовка машины k8s-admin

На машине администратора стоит ansible, все плейбуки необходимо запускать из под пользователя user, все необходимые файлы(inventory, playbook, cert,key) находятся
в каталоге /home/certs-update.

### Установка необходимых компонентов 

Для работы с kubernetes установим дополнительные модули (из под user), модули устанавливаемые с помощь pip также ставим на машину администратора( с нее выполняются все оперции связанные с kubernetes)

```bash

ansible-galaxy collection install kubernetes.core

pip3 install openshift pyyaml kubernetes

```
После установки всего необходимого надо перекинуть ssh ключи по машинам( ключи перекидываем в обе стороны с админской машины на рабочую и с рабочей на админску, чтобы избежать ошибок ),
на всех рабочих машинах с сертификатами генерируем ключ пользователя root, выводим его на экран /root/.ssh/id_rsa.pub, копируем в файл /home/user/.ssh/authorized_keys на машине админа (на машине админа генерируем ключ пользователю user)
выводим ключ на машине админа /home/user/.ssh/id_rsa.pub и копируем в файл каждой рабочей машины /root/.ssh/authorized_keys.

```bash

Для генерации ключей пользователям используем

ssh-keygen 

Для вывода ключей используем

cat /PATH/id_rsa.pub

```

Когда все ключи прокинуты нужно собрать inventory файл в формате yaml, в нем мы указываем имена машин и их адреса и области hosts, которые будут использоваться в плейбуке, а также в области vars указываем перменные для подключения( пользователь и путь к ключу)

###### Inventory file

```bash

all:
  vars:
    ansible_user: root
    ansible_public_ssh_key_path: /home/user/.ssh/authorized_keys
  hosts:
    dev-postgres:
      ansible_host: 192.168.1.71
      ip: 192.168.1.71
      access_ip: 192.168.1.71
    dev-mongo:
      ansible_host: 192.168.1.69
      ip: 192.168.1.69
      access_ip: 192.168.1.69
    dev-minio:
      ansible_host: 192.168.1.81
      ip: 192.168.1.81
      access_ip: 192.168.1.81
    k8s-proxy:
      ansible_host: 192.168.1.79
      ip: 192.168.1.79
      access_ip: 192.168.1.79
  children:
    proxy:
      hosts:
        k8s-proxy:
    postgres:
      hosts:
        dev-postgres:
    minio:
      hosts:
        dev-minio:
    mongo:
      hosts:
        dev-mongo:

```

Проверим работу ключей и inventory

```bash

ansible -i inventory-hosts.yaml all -m ping

```

Пинг прошел успешно, все работает, идем далее

```bash

k8s-proxy | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
dev-minio | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
dev-postgres | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
dev-mongo | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}



```

После надо собрать плейбук(файл в формате yaml) и в каталог /home/certs-update/certs поместить новые ключ и сертификат с именами privkey.pem и fullchain.pem соответственно

###### Playbook

```bash

---
- name: Update certs for proxy
  hosts: proxy

  tasks:


  - name: Update certfiles
    copy:
      src: '/home/certs-update/certs/{{item}}'
      dest: '/etc/nginx/certs/'
    loop:
      - fullchain.pem
      - privkey.pem

  - name: Nginx reload
    ansible.builtin.shell: nginx -t && nginx -s reload && systemctl status nginx
    register: nginx_result

  - name: Output status nginx
    debug: var=nginx_result.stdout_lines

- name: Update certs for kubernetes
  hosts: localhost


  tasks:

  - name: Delete old secret
    kubernetes.core.k8s:
      state: absent
      api_version: v1
      kind: Secret
      namespace: '{{item}}'
      name: tls-secret
    loop:
      - simple-services
      - store
      - monolith
      - monitoring
      - kubernetes-dashboard
      - redis
      - kafka
      - prosody

  - name: Create new secret
    ansible.builtin.shell: kubectl create secret -n '{{item}}' tls tls-secret --cert=/home/certs-update/certs/fullchain.pem --key=/home/certs-update/certs/privkey.pem
    loop:
      - simple-services
      - store
      - monolith
      - monitoring
      - kubernetes-dashboard
      - redis
      - kafka
      - prosody

  - name: TLS Secrets
    ansible.builtin.shell: kubectl get secret -A | grep tls-secret
    register: tls_secrets

  - name: Output TLS Secrets
    debug: var=tls_secrets.stdout_lines


- name: Update certs for postgres
  hosts: postgres

  tasks:

  - name: Update certfiles
    copy:
      src: '/home/certs-update/certs/{{item}}'
      dest: '/etc/postgresql/14/main/certs/'
      owner: postgres
      mode: '0600'
    loop:
      - fullchain.pem
      - privkey.pem



  - name: Postgresql reload
    ansible.builtin.shell: systemctl reload postgresql && systemctl status postgresql
    register: postgres_result

  - name: Output status postgresql
    debug: var=postgres_result.stdout_lines


- name: Update certs for mongo
  hosts: mongo

  tasks:

  - name: Update certfiles
    copy:
      src: '/home/certs-update/certs/{{item}}'
      dest: '/etc/ssl/mongo-certs/'
    loop:
      - fullchain.pem
      - privkey.pem

  - name: Create mongodb certfile
    ansible.builtin.shell: cat /etc/ssl/mongo-certs/privkey.pem /etc/ssl/mongo-certs/fullchain.pem > /etc/ssl/mongo-certs/mongodb.pem

  - name: Permissions for certfile
    ansible.builtin.file:
      path: '/etc/ssl/mongo-certs/mongodb.pem'
      owner: mongodb
      mode: '0600'


  - name: Mongo restart
    ansible.builtin.shell: systemctl restart mongod && systemctl status mongod
    register: mongodb_result

  - name: Output status mongodb
    debug: var=mongodb_result.stdout_lines


- name: Update certsfile for minio
  hosts: minio

  tasks:


  - name: Update certfiles
    copy:
      src: '/home/certs-update/certs/{{item.src}}'
      dest: '/etc/minio/certs/{{item.dest}}'
    loop:
      - { src: fullchain.pem, dest: public.crt }
      - { src: privkey.pem, dest: private.key }

  - name: Minio restart
    ansible.builtin.shell: systemctl restart minio && sleep 3 && systemctl status minio
    register: minio_result

  - name: Output status minio
    debug: var=minio_result.stdout_lines

```


Запускается плейбук из под пользователя user командой из каталога /home/certs-update

```bash

ansible-playbook -i inventory-hosts.yaml playbook-cert-update.yaml

```























