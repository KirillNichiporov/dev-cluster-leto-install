# Подготовка виртуальных машин

На всех машинах необходимо настроить статический ip при установке системы, чтобы при рестарте не был выделен другой адрес.
Если на машинах включен брандмауэр,то необходимо открыть порты (postgres 5432, mariadb 3306, mongodb 27017)

```bash

ufw allow port

```
- db-mongo-dev (ip: 192.168.1.69 dns: db-mongo-dev.smartsafeschool.com)
- db-maria-dev (ip: 192.168.1.70 dns: db-maria-dev.smartsafeschool.com)
- db-postgre-dev (ip: 192.168.1.71 dns: db-postgre-dev.smartsafeschool.com)

Также установить некоторые утилиты для удобства работы:

```bash

sudo apt install mc net-tools

```

# Установка Postgresql

Необходимо выполнить команды по установке, обновляем список пакетов, устанавливаем postgresql, старутем сервис и ставим на автозагруску приперезапуске системы.

```bash



sudo apt update

sudo apt install postgresql postgresql-contrib

sudo systemctl start postgresql.service

sudo systemctl enable postgresql.service

```

Так как при установке системы порты открыты были, то открывать их вручную нет необходимости.

Далее переходим к конфигурационному файлу,настроим сервис на доступ из вне.

```bash

nano /etc/postgresql/14/main/postgresql.conf

```

внутри файла указываем 

```bash

listen_addresses = '*'
port = 5432

```

Рестартуем сервис

```bash

sudo systemctl restart postgresql.service

```

После рестартов можете проверять сервис командами

```bash

sudo systemctl status postgresql.service

и для проверки порта

sudo netstat -ltpn | grep 5432

```

Создадим пользователя admin и настроим для него доступ, переключаемся на пользователя postgres

```bash

sudo -i -u postgres

```

Создаем пользователя, на вопрос о правах суперпользователя отвечаем да указываем имя admin

```bash

createuser --interactive

или без входа в учетку postgres

sudo -u postgres createuser --interactive

```

Создаем базу для пользователя, важно,чтоб база имела такое же название

```bash

createdb admin

или без входа в учетку postgres

sudo -u postgres createdb admin

```


Чтобы работать от имени admin также необходимо создать пользователя linux с таким же именем

```bash

sudo adduser admin

```

Переключаемся на admin, заходим в psql и задаем пароль для доступа к бд

```bash

su - admin
psql
alter user admin with password '********';

```

Пользователь настроен, теперь дадим ему доступ для удаленного подключения, а после настроим tls.

Открываем файл pg_hba.conf

```bash

nano /etc/postgresql/14/main/pg_hba.conf  

```

Внизу прописываем строку аутентификации по паролю и пул необходимых адресов

```bash

host    all             admin           192.168.1.0/24          md5

```

Рестартуем сервис

```bash

sudo systemctl restart postgresql.service

```

### Настройка TLS

Для удобства в каталоге /etc/postgesql/14/main создадим каталог certs сюда поместим ключ и сертификат (privkey.pem и fullchain.pem),
теперь главное присвоить им разрешения 600 иначе postgresql их не подтянет и сделать влядельцем каталога и файлов postgres

```bash
 
chmod 600 /etc/postgesql/14/main/certs/privkey.pem
chmod 600 /etc/postgesql/14/main/certs/fullchain.pem

chow postgres /etc/postgesql/14/main/certs/
chow postgres /etc/postgesql/14/main/certs/privkey.pem
chow postgres /etc/postgesql/14/main/certs/fullchain.pem

```

Далее снова открываем postgresql.conf и уставнавливаем значения в строках(если прописываете внизу файла, то необходимо закомментировать старые значения):

```bash

ssl = on
ssl_cert_file = '/etc/postgresql/14/main/certs/fullchain.pem'
ssl_key_file = '/etc/postgresql/14/main/certs/privkey.pem'

```

Открываем файл pg_hba.conf

```bash

nano /etc/postgresql/14/main/pg_hba.conf  

```

Внизу прописываем строку аутентификации по паролю и пул необходимых адресов уже для ssl

```bash

hostssl    all             admin           192.168.1.0/24          md5

```

Рестартуем сервис

```bash

sudo systemctl restart postgresql.service

```

Теперь перейдем на машину с клиентом psql и проверим удаленное подключение

```bash

root@k8s-admin:/home/user# psql -U admin -h db-postgre-dev.smartsafeschool.com
Password for user admin: 
psql (14.7 (Ubuntu 14.7-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

admin=# 

```
Отлично,идет все через SSL, на этом настройка закончена.

# Установка MariaDB

Для начала добавим репозиторий, из которого и установим mariadb последней версии

```bash

sudo apt install -y software-properties-common

sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'

sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] https://mariadb.mirror.liquidtelecom.com/repo/10.6/ubuntu focal main'

```

Собственно устанавливаем

```bash

sudo apt update 

sudo apt install -y mariadb-server mariadb-client

```

Стартуем сервис и ставим на автозагрузку

```bash

sudo systemctl start mariadb

sudo systemctl enable mariadb

```

Для повышения безопасности запускаем, предложат вввести пароль дял пользователя root базы данных, отключаем удаленный вход для root, он будет доступен только с localhost, удаляем анонимных пользователей, удаляем тестову бд, и перезагружаем таблицу привелегий для того, чтобы изменения вступили в силу.

```bash

sudo mysql_secure_installation

```

Далее настроим пользователя для подключения и дадим ему права

```bash

sudo mariadb -u root -p

```
Создаем пользователя и вместо localhost указываем откуда ему можно подключаться и задаем пароль, разрешаем доступ только с ssl, чтобы через него подключиться, нужно будет явно указать параметр ssl

```bash

CREATE USER 'admin'@'localhost' IDENTIFIED BY 'secret_password' REQUIRE SSL;

Даем все привилегии и применяем

GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' REQUIRE SSL;

FLUSH PRIVILEGES;

EXIT;

```

### Настройка TLS

Создаем каталог для сертификатов 

```bash

mkdir /etc/mysql/certs

```

В него помещаем  ключ privkey.pem и сертификат fullchain.pem, каталог и файлы должны принадлежать mysql, а у файлов разрешение 600, иначе mariadb их не примет

```bash

chmod 600 /etc/mysql/certs/privkey.pem
chmod 600 /etc/mysql/certs/fullchain.pem

chow mysql /etc/mysql/certs
chow mysql /etc/mysql/certs/privkey.pem
chow mysql /etc/mysql/certs/fullchain.pem

```

Открываем конфигурационный файл и прописываем путь к ключу и сертификату:

```bash

nano /etc/mysql/mariadb.conf.d/50-server.cnf

ssl_cert = /etc/mysql/certs/fullchain.pem
ssl_key = /etc/mysql/certs/privkey.pem

```
Рестартуем сервис

```bash

sudo systemctl restart mariadb

```

Заходим в mariadb и проверяем работу tls

```bash

sudo mariadb

SHOW GLOBAL VARIABLES LIKE 'have_ssl';

```

Получаем такой вывод

```bash

+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| have_ssl      | YES   |
+---------------+-------+
1 row in set (0.001 sec)

```

На этом установка и настройка закончены


# Установка MongoDB

Установим репозиторий

```bash

wget -nc https://www.mongodb.org/static/pgp/server-6.0.asc 

cat server-6.0.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/mongodb.gpg >/dev/null 

sudo sh -c 'echo "deb [ arch=amd64,arm64 signed-by=/etc/apt/keyrings/mongodb.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" >> /etc/apt/sources.list.d/mongo.list' 

```

Обновим пакеты и установим mongodb

```bash

sudo apt update 

sudo apt install mongodb-org 

```

Стартуем сервис и ставим на автозагрузку

```bash

sudo systemctl start mongod

sudo systemctl enable mongod  

```

Создадим пользователя и привяжем его к базе admin

```bash

mongosh

используем базу admin

use admin

Создаем пользователя задавая ему имя и пароль


db.createUser({
  user: "admin_user",
  pwd: "password",
  roles: [ { role: "root", db: "admin" } ]
}) 

```

Открываем конфигурационный файл, включаем аутентификацию и рестартуем сервис 

```bash

sudo nano /etc/mongod.conf 

security:
  authorization: enabled

также для доступа из вне настроим

net:
  bindIp: 0.0.0.0
  
sudo systemctl restart mongod

```

### Настройка TLS

Создаем каталог для сертификатов 

```bash

mkdir /etc/ssl/mongo_certs

```

В него помещаем  ключ privkey.pem и сертификат fullchain.pem, и создаем общий файл с ключом и сертификатом, каталог и общий файл должны принадлежать mongod, а у файла разрешение 600, иначе mongod его не примет

```bash

cat /etc/ssl/mongo_certs/privkey.pem /etc/ssl/mongo_certs/fullchain.pem > /etc/ssl/mongo_certs/mongodb.pem

chmod 600 /etc/ssl/mongo_certs/mongodb.pem

chow mongod /etc/ssl/mongo_certs

```
переходим снова в конфигурационный файл и прописываем в нем, доступ удаленный настраиваем только через tls, приудаленном подключении нужно будет явно указываеть параметр ssl

```bash

net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongo-certs/mongodb.pem

```

Рестартуем сервис 

```bash

sudo systemctl restart mongod

```
На этом настройка и установка закончены

# Скрипты для dump и restore

### Postgresql

Был создан каталог /home/admin/backups в нем хранятся дампы, в каталоге /home/admin хранятся скрипты нужно сделать пользователя admin владельцем каталога с бекапами и владельцем скриптов.

###### Dump

```bash

#!/bin/bash

mkdir /home/admin/backups/`date +"%m-%d-%y"`
backup=/home/admin/backups/`date +"%m-%d-%y"`
databases=$(psql -c "SELECT datname FROM pg_database;" | cut -d  ' ' -f 2 | awk '2<FNR' | head -n -2)
data=$(date +'%e.%m.%Y' | awk '{print $1}')

for database in $databases
do
pg_dump $database > $backup/$database-$data.dump
done

```

###### Restore

```bash

#!/bin/bash

backup=/home/admin/backups
database=test_user_long_database
date=03-26-23
dump=test_user_long_database-26.03.2023.dump

psql -U admin -d $database < $backup/$date/$dump

```

### Mariadb

Был создан каталог /home/admin/backups в нем хранятся дампы, в каталоге /home/admin хранятся скрипты

###### Dump

```bash

#!/bin/bash

mkdir /home/admin/backups/`date +"%m-%d-%y"`
backup=/home/admin/backups/`date +"%m-%d-%y"`
databases=$(mariadb -u root -e 'show databases;' | cut -d  ' ' -f 1 | awk '1<FNR')
data=$(date +'%e.%m.%Y' | awk '{print $1}')

for database in $databases
do
mysqldump $database > $backup/$database-$data.dump
done

```

###### Restore

```bash

#!/bin/bash

backup=/home/admin/backups
database=mysql
date=03-26-23
dump=mysql-26.03.2023.dump

mysql -u root $database < $backup/$date/$dump

```

### MongoDB

Был создан каталог /home/admin/backups в нем хранятся дампы, в каталоге /home/admin хранятся скрипты

###### Dump

```bash

#!/bin/bash

backup=/home/admin/backups/`date +"%m-%d-%y"`
host=localhost.smartsafeschool.com
password=$(cat /home/admin/passfile | base64 --decode)
mongodump -u admin-user -p $password --host $host --out $backup --tlsInsecure --ssl > /home/admin/cr.log 2>&1

```

###### Restore

```bash

#!/bin/bash

database=tesdb
host=localhost.smartsafeschool.com
password=$(cat /home/admin/passfile | base64 --decode)
mongorestore --authenticationDatabase=admin  -u admin-user -p $password --host $host -d $database backups/03-24-23/$database/ --ssl > /home/admin/cf.log 2>&1

```
