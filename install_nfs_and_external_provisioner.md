# Настройка машины для NFS

Для при установке системы необходимо задать статический ip, дабы, чтобы клиент для nfs в kubernetes не потерял сервер nfs после рестарта,
ибо dhcp может выделить другой адрес.
Также необходимо открыть порты для nfs, если используется брандмауэр.

```bash

sudo ufw allow 2049
sudo ufw allow 111

```

Установим все необходимые утилиты для работы

```bash

sudo apt install nfs-kernel-server mc net-tools

```

Проверим порты

```bash

rpcinfo -p | grep nfs

100003    3   tcp   2049  nfs
100003    4   tcp   2049  nfs


```

Также важно проверить поддерживается ли NFS на уровне ядра:

```bash

cat /proc/filesystems | grep nfs
 
Видим, что работает, но если нет, нужно вручную загрузить модуль ядра nfs

вывод: nodev	nfsd

Для загрузки в ядро

modprobe nfs

```

Ставим сервис на автозагрузку

```bash

sudo systemctl enable nfs-server

```

Создаем каталог для хранения 

```bash

mkdir -p /var/nfs/monolith

```

Важно создаем пользователя и присваиваем ему uid 1001,также делаем владельцем созданного каталога,так как кластер разворачивался не через root, то при создании pvc для хранения будет отказано в доступе, поэтому учитываем этот момент заранее.

```bash

sudo adduser anon

usermod -u 1001 anon

chown anon /var/nfs/monolith

```bash

Переходим уже к настройке nfs, сделаем доступ к папке, для того чтобы кластер мог получить доступ ко всем файлам можно попросить NFS все запросы считать запросами от анонимного пользователя, а анонимному пользователю присвоить UID 1001, для этого был создан пользователь.

Переходим в файл /etc/exports

```bash

nano /etc/exports

И там прописываем

/var/nfs/monolith 192.168.1.0/24(rw,sync,all_squash,root_squash,anonuid=1001,anongid=1001)

```

- rw - разрешить чтение и запись в этой папке;
- ro - разрешить только чтение;
- sync - отвечать на следующие запросы только тогда, когда данные будут сохранены на диск (по умолчанию);
- async - не блокировать подключения пока данные записываются на диск;
- secure - использовать для соединения только порты ниже 1024;
- insecure - использовать любые порты;
- nohide - не скрывать поддиректории при, открытии доступа к нескольким директориям;
- root_squash - подменять запросы от root на анонимные, используется по умолчанию;
- no_root_squash - не подменять запросы от root на анонимные;
- all_squash - превращать все запросы в анонимные;
- subtree_check - проверять не пытается ли пользователь выйти за пределы экспортированной папки;
- no_subtree_check - отключить проверку обращения к экспортированной папке, улучшает производительность, но снижает безопасность, можно использовать когда экспортируется раздел диска;
- anonuid и anongid - указывает uid и gid для анонимного пользователя.

Root остается анонимным, так безопаснее, если отключить, то любой  root может получить доступ на чтение и запись.

Когда все настроено, обновляем таблицу экспорта nfs


```bash

sudo exportfs -a


```


Теперь переходим на воркер-машины и устанавливаем клиенты для nfs, без них кластер попросту не подключится к nfs.

```bash

sudo apt install nfs-common cifs-utils

```

Переходим на машину администратора, на ней надо установить helm репозиторий для nfs-subdir-external-provisioner, и обновим чарты

```bash

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm repo update

```

Теперь установим provisioner через helm, необходимо указать адрес к nfs каталог и название storageclass

```bash

helm install nfs-monolith-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.1.80 --set nfs.path=/var/nfs/monolith --set storageClass.name=monolith-storage

```

Проверим статус

```bash

kubectl get pods -n default


NAME                                                              READY   STATUS    RESTARTS   AGE
nfs-monolith-subdir-external-provisioner-nfs-subdir-extern2prvm   1/1     Running   0          67m

```

Отлично, под запущен

Теперь сделаем pvc хранилку для monolith

Создаем namespace monolith

```bash

kubectl create namespace monolith 

```

Далее создаем файл filename.yaml для pvc и помещаем в него манифест, accesMode ставим на чтение и запись многими узлами, а так же выделяем место и указываем storageclass

```bash

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: monolith-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "monolith-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi

```

Добавляем в кластер, важно использовать нужный nemspace, куда будет помещен сервис, чтоб он потом нашел pvc

```bash

kubectl apply -f filename.yaml -n monolith

```
Проверяем


```bash

kubectl get pvc -n monolith

NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
monolith-pvc   Bound    pvc-60d7406b-c5bb-4a0e-ad57-41c2747c7175   50Gi       RWX            monolith-storage   2m56s

```
Статус bound это хорошо, pvc работает, на этом настройка nfs и интеграция с кластером закончена






