# Установка Nightscout на CentOS8
Установим Nightscout на свой сервер под управлением CentOS 8. Предполагается, что вы знакомы с командной строкой и работой с Linux.

Имеем установленный сервер под управлением CentOS 8, доступ по ssh. Для простоты SELinux отключен. Вход в консоль осуществлен под пользователем ```root``` или пользователя с правами ```sudo```. Сервер должен иметь "белый" статический IP или находиться за NAT-ом, тоже со статическим IP. Так же, должно быть зарегистрировано имя с указанием в DNS на наш сервер. Для примера, буду использовать имя "night.domain.ru". Сделаем необходимую подготовку, установим программы:
```bash
dnf install git nano mc atop htop -y
dnf groupinstall 'Development Tools' -y
```
### Установим MongoDB

Официальная документация по установке доступна [тут](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/)

Добавим репозиторий
```bash
echo '
[mongodb-org-4.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
' > /etc/yum.repos.d/mongodb-org-4.4.repo
```
Устанавливаем
```bash
dnf install mongodb-org -y
```
Делаем автозапуск службы, запускаем MongoDB и проверяем работу
```bash
systemctl enable mongod.service
systemctl start mongod.service
systemctl status mongod.service
```
Должно быть примерно так:
```bash
● mongod.service - MongoDB Database Server
   Loaded: loaded (/usr/lib/systemd/system/mongod.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2020-09-06 12:28:55 +04; 19min ago
     Docs: https://docs.mongodb.org/manual
 Main PID: 9384 (mongod)
   Memory: 74.5M
   CGroup: /system.slice/mongod.service
           └─9384 /usr/bin/mongod -f /etc/mongod.conf
```
Добавим пользователя ```admin``` и включаем аутентификацию. Официальная [документация](https://docs.mongodb.com/manual/tutorial/enable-authentication/). Для этого заходим в консоль mongo:
```bash
mongo
```
и вставляем следующий код (замените ```ADMIN_PASSWORD``` на свой пароль):
```bash
> use admin
> db.createUser({user: "admin",pwd: "ADMIN_PASSWORD",roles: [{ role: "userAdminAnyDatabase", db: "admin" }, { role: "readWriteAnyDatabase", db: "admin" }, { role: "root", db: "admin" }, "readWriteAnyDatabase" ] })
> quit()
```
В ответ вы должны получить такое: ```Successfully added user```. Включаем аутентификацию, для этого редактируем файл ```/etc/mongod.conf```, находим в нем строку ```#security:``` и приводим к такому виду:
```bash
security:
  authorization: "enabled"
```
и перезапускаем MongoDB
```bash
systemctl restart mongod.service
```
Создаем базу(```nightscout```), добавляем пользователя(```userdb```) и пароль(```passdb``` - заменить на свой) к ней. Заходим в консоль монго
```bash
mongo -u admin -p --authenticationDatabase admin
вводим пароль admin
```
Выполняем 
```bash
> use nightscout
> db.createUser({user: "userdb", pwd: "passdb", roles:["readWrite"]})
> db.createCollection("entries")
> quit()
```
Проверяем
```bash
> use nightscout
> show dbs
```
Должны увидеть
```bash
admin       0.000GB
config      0.000GB
local       0.000GB
nightscout  0.000GB
```
База готова. Выходим из консоли MongoDB ```Ctrl+C```. Экспортирование базы из Heroku будет написано ниже.

### Установим NodeJS

Ставить будем из репозитория **AppStream**, посмотрим существующие версии
```bash
dnf module list nodejs
```
Должно быть примерно так:
```bash
CentOS-8 - AppStream
Name           Stream         Profiles                                      Summary                  
nodejs         10 [d]         common [d], development, minimal, s2i         Javascript runtime       
nodejs         12             common [d], development, minimal, s2i         Javascript runtime       
```
Доступны два потока, 10 и 12. **[d]** указывает, что версия 10 — это поток по умолчанию. Изменим его на 12.
```bash
dnf module enable nodejs:12 -y
```
Установим NodeJS
```bash
dnf install nodejs -y
```
Вместе с NodeJS устанавливается ```npm``` - пакетный менеджер, Проверим версии
```bash
node --version && npm --version
```
У меня такие: v12.18.3, 6.14.6, у вас могут отличаться

### Устанавливаем Nightscout

Nightscout не может работать от ```root```, по этому создадим пользователя и сделаем запуск от него. Создаем пользователя ```nightscout```, определяя его домашний каталог ```/opt/nightscout```
```bash
useradd -d /opt/nightscout -m -c "User for nightscout" nightscout
```
Заходим под новым пользователем и устанавливаем Nightscout
```bash
su - nightscout
git clone https://github.com/nightscout/cgm-remote-monitor.git
cd cgm-remote-monitor
npm install
```
Создаем исполняемый файл ```/opt/nightscout/cgm-remote-monitor/start.sh``` следующего содержания. 
Обратите внимание на:
```MONGO_CONNECTION``` - параметры для подключения к MongoDB.
```API_SECRET``` - секретный ключ для доступа к сайту.
Остальные параметры можно посмотреть [тут](https://github.com/nightscout/cgm-remote-monitor#environment)

```bash
#!/bin/bash

# environment variables
export MONGO_CONNECTION="mongodb://userdb:passdb@localhost:27017/nightscout"
export DISPLAY_UNITS="mmol"
export BASE_URL="http://night.domain.ru"
export PORT=1337
export DEVICESTATUS_ADVANCED="true"
export mongo_collection="entries"
export API_SECRET="1234567890AB"
export ENABLE="careportal basal rawbg cob iob cage bwp upbat sage pump"
export TIME_FORMAT=24
export THEME=colors
export LANGUAGE=ru
export SCALE_Y=linear
export HOSTNAME=0.0.0.0

# start server
node /opt/nightscout/cgm-remote-monitor/server.js
```
Сделаем наш файл исполняемым
```bash
chmod +x start.sh
```
Так же, уберем возможность запускать файл ```setup.sh``` - он нам не нужен, но находится рядом.
```bash
chmod -x setup.sh
```
и пробуем запустить.
```bash
./start.sh
```
В консоль начало приходить примерно такое:
```bash
Load Complete:
	 
reloading sandbox data
all buckets are empty
For the COB plugin to function you need a treatment profile
For the Basal plugin to function you need a treatment profile
WS: running websocket.update
WS: emitted clear_alarm to all clients
tick 2020-09-07T18:29:33.800Z
```
Останавливаем ```Ctrl+C```

Дальнейшие работы проводим под пользователем ```root```, нажимаем ```Ctrl+D``` и возвращаемся в консоль рута.
Делаем службу, чтоб Nightscout запускался автоматически, создаем файл ```/etc/systemd/system/nightscout.service``` следующего содержания
```bash
[Unit]
Description=Nightscout Service
After=network.target
After=mongod.service

[Service]
Type=simple

User=nightscout
Group=nightscout

WorkingDirectory=/opt/nightscout/cgm-remote-monitor
ExecStart=/opt/nightscout/cgm-remote-monitor/start.sh

[Install]
WantedBy=multi-user.target
```
перезагружаем демона и стартуем сервис
```bash
systemctl daemon-reload
systemctl enable nightscout.service
systemctl start nightscout.service
```
Через некоторое время проверяем, что все работает:
```bash
systemctl status nightscout.service
● nightscout.service - Nightscout Service
   Loaded: loaded (/etc/systemd/system/nightscout.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-09-07 22:45:25 +04; 18s ago
 Main PID: 37456 (start.sh)
    Tasks: 12 (limit: 12525)
   Memory: 61.0M
   CGroup: /system.slice/nightscout.service
           ├─37456 /bin/bash /opt/nightscout/cgm-remote-monitor/start.sh
           └─37458 node /opt/nightscout/cgm-remote-monitor/server.js
```
Проверим, что порт слушается и проверим работу сайта
```bash
netstat -ltupen | grep 1337
tcp        0      0 0.0.0.0:1337            0.0.0.0:*               LISTEN      0          123534     37458/node          
```

```bash
curl http://localhost:1337 | grep "<title>"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 43706  100 43706    0     0  3283k      0 --:--:-- --:--:-- --:--:-- 3556k
      <title>Nightscout</title>
```
Если увидели нечто такое - сайт работает.

### Устанавливаем Nginx

Установим сам nginx и certbot (для получения сертификата от **Let’s Encrypt**)
```bash
dnf install nginx certbot python3-certbot-nginx -y
```
Откроем файл ```/etc/nginx/nginx.conf``` и закомментируем секцию ```server```, т.е. приведем к такому виду
```bash
#    server {
#        listen       80 default_server;
#        listen       [::]:80 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }
```
Создаем новый каталог и файлы с конфигурацией
```bash
mkdir /etc/nginx/includes
```
файл ```/etc/nginx/includes/ssl```
```bash
#ssl_certificate	/etc/pki/tls/certs/fullchain.pem;
#ssl_certificate_key	/etc/pki/tls/certs/privkey.pem;

ssl_protocols TLSv1.2 TLSv1.3;

ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
ssl_prefer_server_ciphers on;

# HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; ";

# OCSP Stapling ---
# fetch OCSP records from URL in ssl_certificate and cache them
ssl_stapling on;
ssl_stapling_verify on;

ssl_trusted_certificate /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem;

resolver 8.8.8.8 valid=300s;
resolver_timeout 5s;
```
файл ```/etc/nginx/includes/proxy_pass_reverse```
```bash
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Server $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
Создадим файл с конфигурацией нашего сайта ```/etc/nginx/conf.d/night.conf```
```bash
server {
    listen  80;
    server_name night.domain.ru default_server;
    # enforce https
    return 301 https://$server_name$request_uri;
}
server {
    listen 443 ssl http2;
    server_name night.domain.ru default_server;

    access_log /var/log/nginx/night-access.log;
    error_log /var/log/nginx/night-error.log;

    include /etc/nginx/includes/ssl;

    location / {
        proxy_pass http://127.0.0.1:1337/;
        include /etc/nginx/includes/proxy_pass_reverse;
    }
}
```
Для получения сертификата, надо изменить настройки nginx, для этого создаем каталог
```bash
mkdir -p /var/www/letsencrypt
chown -R nginx:nginx /var/www/letsencrypt
```
Теперь нам нужно сделать так, чтобы любой запрос вида: ```http://night.domain.ru/.well-known/acme-challenge``` приводил к физическому размещению ```/var/www/letsencrypt/.well-known/acme-challenge```, для этого создадим файл ```/etc/nginx/includes/letsencrypt``` такого содержания
```bash
location ^~ /.well-known/acme-challenge/ {
   default_type "text/plain";
   root /var/www/letsencrypt;
}
location = /.well-known/acme-challenge/ {
   return 404;
}
```
Теперь внесем изменения в файл ```/etc/nginx/conf.d/night.conf```, добавив в него строку ```include /etc/nginx/includes/letsencrypt;```, должно получится так: (лишние строки я обрезал)
```bash
server {
    listen 443 ssl http2;
..........
    include /etc/nginx/includes/ssl;
    include /etc/nginx/includes/letsencrypt;
..........
}
```
Теперь можно сделать запрос на выпуск сертификатов. **Обращаю внимание**, что у вас должна быть правильно настроена запись в DNS, при которой все запросы по имени ```night.domain.ru``` должны приходить на ваш сервер. Проверить это можно через команду 
```bash
host night.domain.ru
night.domain.ru has address 123.45.67.89
```
,где ```123.45.67.89``` - IP вашего сервера
Так же, необходимо открыть порты 80, 443 на фаерволе. Я использую iptables (не люблю firewalld), по этому добавил правила:
```bash
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
```
Теперь можно сделать запрос на создание сертификата:
```bash
# Запрос без оповещения на e-mail
certbot certonly --nginx -d night.domain.ru --register-unsafely-without-email
# ИЛИ такой, с оповещением на my_email@domain.ru
certbot certonly --nginx -d night.domain.ru -m my_email@domain.ru
```
В результате, вы должны увидеть такой лог (обрезан):
```bash
.........
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/night.domain.ru/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/night.domain.ru/privkey.pem
.........
```
тут нас интересует пути до сертификата и приватный ключ. Изменим файл ```/etc/nginx/includes/ssl```, в самом начале:
```bash
ssl_certificate		/etc/letsencrypt/live/night.domain.ru/fullchain.pem;
ssl_certificate_key	/etc/letsencrypt/live/night.domain.ru/privkey.pem;
```
теперь настроим службы nginx и запустим его
```bash
systemctl enable nginx.service
systemctl start nginx.service
```
Теперь можно зайти на сайт ```https://night.domain.ru``` и продолжить настройки самого сайта.
Так же, можно проверить на правильную настройку nginx и сертификатов на этом [сайте](https://www.ssllabs.com/ssltest).

Настроим автообновление сертификата. Для этого выполним команду 
```bash
export EDITOR=mcedit
crontab -e
```
и добавим строку в самый конец:
```bash
0 0,12 * * * python3 -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot -q renew --renew-hook "systemctl restart nginx.service"
```

### Импортируем данные с Heroku
Заходим на [сайт](https://dashboard.heroku.com/apps/mytestgsm/settings), нажимаем кнопку "Reveal Config Vars", находим переменную "MONGODB_URI". В ней содержится вся необходимая информация для создания дампа. В моем случае она такая (данные изменены):
```bash
mongodb://heroku_slllg:4bo58b2jud67tjqvime2s@ds123963.mlab.com:23963/heroku_slllg
```
выполняем команды
```bash
cd ~
# Дамп с Heroku
mongodump --host="ds123963.mlab.com:23963" -d heroku_slllg --username="heroku_slllg" --password="4bo58b2jud67tjqvime2s"
# Восстановление в нашу базу
mongorestore -d nightscout --username="userdb" --password="passdb" dump/heroku_slllg
```
Заходим на сайт и видим наши данные.
