# Ingress-controller

Настроем ингресс контроллер, на внешний трафик,чтобы он не заменял адрес клиента на свой

Открываем файл командой

```bash

kubectl edit service -n ingress-nginx ingress-nginx-controller

```

И выставляем параметр 

```bash

externalTrafficPolicy: Local

```

После редактируем все ингресс-манифесты магазина, чтобы в сервис попал реальный ip клиента

Вносим в поле annotations запись и адрес меняем на адрес прокси-сервера

```bash

    nginx.ingress.kubernetes.io/server-snippet: |
       set_real_ip_from 192.168.1.79;
        real_ip_header X-Real-IP;


```

Обновляем все ингрессы, в папке магазина на машине администратора лежит скрипт

# Proxy-server

Для получения адреса от прокси сервера необходимо внести изменения в файл /etc/nginx/sites-enable/ingress  в поле location

```bash

proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

```

Далее проверяем настройки и загружаем изменения

```bash

nginx -t

nginx -s reload

```

Если изменения не применились,то можно рестаратнуть nginx 

```bash

systemctl restart nginx

```


