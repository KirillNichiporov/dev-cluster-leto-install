# Подготовка виртуальных машин

На всех машинах необходимо настроить статический ip при установке системы, чтобы при рестарте не был выделен другой адрес.

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

Теперь настроим tls

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

