# Установка Nightscout на CentOS8
Установим Nightscout на свой сервер под управлением CentOS8. Предполагается, что вы знакомы с командной строкой и работой с Linux.

Имеем установленный сервер под управлением CentOS8, для простоты SELinux отключен. Сделаем необходимую подготовку, установим необходимые программы:
```bash
dnf install gcc git -y
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

