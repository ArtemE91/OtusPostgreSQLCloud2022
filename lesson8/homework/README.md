## Домашнее задание 8 Занятие


### Разворачиваем и настраиваем БД с большими данными


Необходимо провести сравнение скорости работы запросов на различных СУБД

* Выбрать одну из СУБД
* Загрузить в неё данные (от 10 до 100 Гб)
* Сравнить скорость выполнения запросов на PosgreSQL и выбранной СУБД
* Описать что и как делали и с какими проблемами столкнулись

Решил протестировать разницу на боевой таблице в Pet-Project. Скопируем ее на хост. Размер файла в *.csv 10.35GB
В таблице хранится информация по частотности поисковых запросах и другая необходимая информация.

Скинем файл в облако
```shell
$ scp file.csv user@51.250.110.32:~/datasource/ 
```

Установим postgres на систему и проверим что кластер запущен
```shell
$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
$ sudo -u postgres pg_lsclusters  
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

Установим Mongo и запустим
```shell
$ curl -fsSL https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
OK
$ echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
$ sudo apt update
$ sudo apt install mongodb-org
$ sudo systemctl status mongod
```

Подключимся к Postgres и загрузим данные:
```sql
create database otus;
create table "taginfo";
```

Подключимся к mongo и загрузим данные:
```shell
$ mongo
> use otus
switched to db otus
> db.createCollection("taginfo")
{ "ok" : 1 }
> exit

```

mongoimport --db otus --collection taginfo --file taginfo_mongo_0.json --jsonArray