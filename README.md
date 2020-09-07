# Установка Nightscout на CentOS8
Установим Nightscout на свой сервер под управлением CentOS8. Предполагается, что вы знакомы с командной строкой и работой с Linux.

Имеем установленный сервер под управлением CentOS8, для простоты SELinux отключен. Сделаем необходимую подготовку, установим необходимые программы:
```bash
dnf install git -y
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
# systemctl status mongod.service 
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
> db.createUser({user: "admin",pwd: "ADMIN_PASSWORD",roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]})
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
Должны увидить
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
У меня такие: v12.18.3, 6.14.6

### Устанавливаем Nightscout

Ставить будем в папку ```/opt/nightscout```
```bash
mkdir /opt/nightscout
cd /opt/nightscout
git clone https://github.com/nightscout/cgm-remote-monitor.git
cd cgm-remote-monitor
npm install
```



Создаем запускной файл ```/opt/nightscout/cgm-remote-monitor/start.sh``` следующего содержания. Обратите внимание на ```MONGO_CONNECTION``` - тут указываем параметры для подключения к MongoDB. Остальные параметры можно посмотреть [тут](https://github.com/nightscout/cgm-remote-monitor#environment)
```bash
#!/bin/bash

# environment variables
export MONGO_CONNECTION="mongodb://userdb:passdb@localhost:27017/nightscout"
export DISPLAY_UNITS="mmol"
export BASE_URL="http://my_site.ru"
export PORT=1337
export DEVICESTATUS_ADVANCED="true"
export mongo_collection="entries"
export API_SECRET=NIGHTSCOUT_API_SECRET
export ENABLE="careportal basal rawbg cob iob cage bwp upbat sage pump"
export TIME_FORMAT=24
export THEME=colors
export LANGUAGE=ru
export SCALE_Y=linear
export HOSTNAME=0.0.0.0

# start server
node /opt/nightscout/cgm-remote-monitor/server.js
```
Сделаем наш файл запускным
```bash
chmod +x start.sh
```
и пробуем запустить.
```bash
./start.sh
```
Почему-то у меня не заработал Nightscout сразу, причиной оказалось
```bash
Error: ENOENT: no such file or directory, open '/opt/nightscout/cgm-remote-monitor/tmp/cacheBusterToken'
```
Чтоб ее исправить, надо создать каталог ```tmp``` и файл ```cacheBusterToken```. Странно, что программа сама не делает этого.
```bash
mkdir /opt/nightscout/cgm-remote-monitor/tmp
touch /opt/nightscout/cgm-remote-monitor/tmp/cacheBusterToken
```
После этого все запустилось
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
Останавливаем ```Ctrl+C``` и делаем службу, чтоб Nightscout запускался автоматически, создаем файл ```/etc/systemd/system/nightscout.service``` следующего содержания
```bash
[Unit]
Description=Nightscout Service      
After=network.target
After=mongod.service

[Service]
Type=simple
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
100 43594  100 43594    0           <title>Nightscout</title>
0  4257k      0 --:--:-- --:--:-- --:--:-- 4257k
```
Если увидили нечто такое - сайт работает.
