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
**(Не обязательный пункт)**

> По умолчанию, любой пользователь на сервере может выполнять любые функции (запросы) в MongoDB. 

Чтобы этого избежать необходимо добавить пользователя ```admin``` и  включить аутентификацию. Для этого заходим в консоль mongo:
```bash
mongo
```
и вставляем следующий код (замените ```ADMIN_PASSWORD``` на свой пароль):
```bash
> use admin
> db.createUser({user: "admin",pwd: "ADMIN_PASSWORD",roles: [{ role: "userAdminAnyDatabase", db: "admin" }]})
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

**Продолжаем**, создаем базу(```nightscout```), добавляем пользователя(```userdb```) и пароль(```passdb``` - заменить на свой!) к ней. Заходим в консоль монго (если делали предыдущий шаг)
```bash
mongo -u admin -p --authenticationDatabase admin
вводим пароль admin (см. предыдущий шаг)
```
или
```bash
mongo
```
Выполняем 
```bash
> use nightscout
> db.createUser({user: "userdb", pwd: "passdb", roles:["readWrite"]})
> quit()
```
База готова. Экспортирование базы из Heroku будет написано ниже.

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
Создаем запускной файл ```start.sh``` следующего содержания, в котором указываем переменные.
Обратите внимание на ```MONGO_CONNECTION``` - тут указываем параметры для подключения к MongoDB. Остальные параметры можно посмотреть [тут](https://github.com/nightscout/cgm-remote-monitor#environment)
```bash
#!/bin/bash

# environment variables
export DISPLAY_UNITS="mg/dl"
export MONGO_CONNECTION="mongodb://userdb:passdb@localhost:27017/nightscout"
export PORT=1337
export API_SECRET="Api_Secret_min_12_symbols"
export PUMP_FIELDS="reservoir battery status"
export DEVICESTATUS_ADVANCED=true
export ENABLE="careportal basal cage sage boluscalc rawbg iob bwp bage mmconnect bridge openaps pump iob maker"
export TIME_FORMAT=24
export BASE_URL="YOURS_INTERNET_URL.RU"
export INSECURE_USE_HTTP=true
export ALARM_HIGH=on
export ALARM_LOW=on
export ALARM_TIMEAGO_URGENT=on
export ALARM_TIMEAGO_URGENT_MINS=30
export ALARM_TIMEAGO_WARN=on
export ALARM_TIMEAGO_WARN_MINS=15
export ALARM_TYPES=simple
export ALARM_URGENT_HIGH=on
export ALARM_URGENT_LOW=on
export AUTH_DEFAULT_ROLES=denied
export BG_HIGH=180
export BG_LOW=72
export BG_TARGET_BOTTOM=90
export BG_TARGET_TOP=162
export BRIDGE_MAX_COUNT=1
export BRIDGE_PASSWORD=
export BRIDGE_SERVER=EU
export BRIDGE_USER_NAME=
export CUSTOM_TITLE=MyTitle
export DISABLE=
export MONGO_COLLECTION=entries
export NIGHT_MODE=on
export OPENAPS_ENABLE_ALERTS=true
export OPENAPS_FIELDS='status-symbol status-label iob meal-assist rssi'
export OPENAPS_RETRO_FIELDS='status-symbol status-label iob meal-assist rssi'
export OPENAPS_URGENT=60
export OPENAPS_WARN=20
#export PAPERTRAIL_API_TOKEN=some_token
export PUMP_ENABLE_ALERTS=true
export PUMP_FIELDS='battery reservoir clock status'
export PUMP_RETRO_FIELDS='battery reservoir clock status'
export PUMP_URGENT_BATT_V=1.3
export PUMP_URGENT_CLOCK=30
export PUMP_URGENT_RES=10
export PUSHOVER=
export SHOW_FORECAST=openaps
export SHOW_PLUGINS='pump iob sage cage careportal'
export SHOW_RAWBG=noise
export THEME=colors

# start server
node server.js
```
