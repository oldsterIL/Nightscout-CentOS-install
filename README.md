# Установка Nightscout на CentOS8
Установим Nightscout на свой сервер под управлением CentOS8. Предполагается, что вы знакомы с командной строкой и работой с Linux.

Имеем установленный сервер под управлением CentOS8, для простоты SELinux отключен.

### Установим MongoDB

Официальная документация по установке доступна тут: https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/

Добавим репозиторий
```
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
```
dnf install mongodb-org -y
```
Делаем автозапуск службы, запускаем MongoDB и проверяем работу
```
systemctl enable mongod.service
systemctl start mongod.service
systemctl status mongod.service
```
Должно быть примерно так:
```
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
(Не обязательный пункт)

По умолчанию, любой пользователь на сервере может выполнять любые функции (запросы) в MongoDB. Чтобы этого избежать необходимо добавить пользователя ```admin``` и  включить аутентификацию. Для этого заходим в консоль mongo:
```
mongo
```
и вставляем следующий код (замените ```ADMIN_PASSWORD``` на свой пароль):
```
use admin
db.createUser(
  {
    user: "admin",
    pwd: "ADMIN_PASSWORD",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
```
В ответ вы должны получить такое: ```Successfully added user```. Выходим из оболочки (нажимаем ```Ctrl+C```). Включаем аутентификацию, для этого редактируем файл ```/etc/mongod.conf```, находим в нем строку ```#security:``` и приводим к такому виду:
```
security:
  authorization: "enabled"
```
и перезапускаем MongoDB
```
systemctl restart mongod.service
```
проверяем
```
mongo -u admin -p --authenticationDatabase admin
вводим пароль
```


Т.к. сервер работает Создаем базу и добавляем пользователя к ней.
```

```
