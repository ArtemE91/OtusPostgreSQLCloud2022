## Домашнее задание 8 Занятие


### Разворачиваем и настраиваем БД с большими данными


Необходимо провести сравнение скорости работы запросов на различных СУБД

* Выбрать одну из СУБД
* Загрузить в неё данные (от 10 до 100 Гб)
* Сравнить скорость выполнения запросов на PosgreSQL и выбранной СУБД
* Описать что и как делали и с какими проблемами столкнулись

Решил протестировать разницу на боевой таблице в Pet-Project. Скопируем ее на хост. Размер файла в *.csv 10.35GB
В таблице хранится информация по частотности поисковых запросах и другая необходимая информация. Тестировать будем
поиск подстроки в строке с сортировкой по частотности.

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
-- При загрузке дампа возникли ошибки. Сделал дамп через pg_dump и все загрузилось без проблем
$ sudo psql -h localhost -d otus -U postgres -f backup_taginfo.sql
```

Подключимся к mongo и загрузим данные:
```shell
$ mongo
> use otus
switched to db otus
> db.createCollection("taginfo")
{ "ok" : 1 }
> exit
-- csv файл не удалось загрузить. Сделал dump с помощью  python manage.py dumpdata в json.
-- Файл пришлось распарссить ijson и сгенирировать 60 файлов по 200 000 записей.
-- Через bash залил все файлы в mongo
$ mongoimport --db otus --collection taginfo --file taginfo_mongo_{index}.json --jsonArray
```

Проверим время выполнения запроса без установки индексов в Mongo
```shell
> db.taginfo.find({'name': {'$regex': 'мяч'}}).explain("executionStats")
"nReturned" : 20128
"executionTimeMillis" : 11205
> db.taginfo.find({'name': {'$regex': 'мяч'}}).limit(5).sort({"frequency_30": 1}).explain("executionStats")
"executionTimeMillis" : 11622
```

Проверим время выполнения запроса без установки индексов в PostgreSQL
```shell
\timing
select name, frequency_30 from taginfo where name like '%мяч%' order by frequency_30 limit 5;
Time: 4483.133 ms (00:04.483)
```

Создадим необходимые индексы в монго
```shell
-- Создание индекса по возрастанию для поля частотности
> db.taginfo.createIndex({frequency_30: 1})
{
	"numIndexesBefore" : 1,
	"numIndexesAfter" : 2,
	"createdCollectionAutomatically" : false,
	"ok" : 1
}
> db.taginfo.find({'name': {'$regex': 'мяч'}}).limit(200).sort({"frequency_30": 1}).explain("executionStats")
"executionTimeMillis" : 466
> db.taginfo.find({$or: [{'name': {$regex: 'мяч'}}, {'name': {'$regex': 'баскетбольный'}}]}).limit(200).sort({"frequency_30": 1}).explain("executionStats")
"executionTimeMillis" : 505
-- создадим индекс для тестового поля name
db.taginfo.createIndex({name : "text"}, {default_language: "russian"})
{
	"numIndexesBefore" : 2,
	"numIndexesAfter" : 3,
	"createdCollectionAutomatically" : false,
	"ok" : 1
}
db.taginfo.find({$or: [{'name': {$regex: 'платье'}}, {'name': {'$regex': 'кофта'}}]}).limit(200).sort({"frequency_30": 1}).explain("executionStats")
"executionTimeMillis" : 494
```

Создадим необходимые индексы в postgresql
```shell
-- Создание индекса по возрастанию для поля частотности
$ CREATE INDEX frequency_30_desc_index ON taginfo (frequency_30 DESC NULLS LAST);
$ select name, frequency_30 from taginfo where name like '%мяч%' order by frequency_30 limit 200;
Time: 4406.017 ms (00:04.406)
$ select name, frequency_30 from taginfo where name like '%мяч%' or name like '%баскетбольный%' order by frequency_30 limit 200;
Time: 7674.655 ms (00:07.675)
-- Поиск 2 слов очень сильно увеличивает время
-- Посмотрим план выполнения
$ explain select name, frequency_30 from taginfo where name like '%мяч%' order by frequency_30 limit 5;
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Limit  (cost=331326.84..331327.43 rows=5 width=61)
   ->  Gather Merge  (cost=331326.84..331443.05 rows=996 width=61)
         Workers Planned: 2
         ->  Sort  (cost=330326.82..330328.06 rows=498 width=61)
               Sort Key: frequency_30
               ->  Parallel Seq Scan on taginfo  (cost=0.00..330318.55 rows=498 width=61)
                     Filter: ((name)::text ~~ '%мяч%'::text)
-- Добавим EXTENTION для разбиение поля name на триграммы
$ CREATE EXTENSION pg_trgm;
$ CREATE INDEX trgm_idx_taginfo_name ON taginfo USING gin (name gin_trgm_ops);
$explain analyze select name, frequency_30 from taginfo where name like '%мяч%' or name like '%баскетбольный%' order by frequency_30 limit 200;
                                                                         QUERY PLAN                                                                         
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=9244.73..9245.23 rows=200 width=61) (actual time=201.551..201.577 rows=200 loops=1)
   ->  Sort  (cost=9244.73..9250.71 rows=2392 width=61) (actual time=201.549..201.561 rows=200 loops=1)
         Sort Key: frequency_30
         Sort Method: top-N heapsort  Memory: 76kB
         ->  Bitmap Heap Scan on taginfo  (cost=251.14..9141.35 rows=2392 width=61) (actual time=154.251..197.447 rows=20281 loops=1)
               Recheck Cond: (((name)::text ~~ '%мяч%'::text) OR ((name)::text ~~ '%баскетбольный%'::text))
               Rows Removed by Index Recheck: 24
               Heap Blocks: exact=8347
               ->  BitmapOr  (cost=251.14..251.14 rows=2392 width=0) (actual time=152.815..152.817 rows=0 loops=1)
                     ->  Bitmap Index Scan on trgm_idx_taginfo_name  (cost=0.00..28.97 rows=1196 width=0) (actual time=4.405..4.405 rows=20132 loops=1)
                           Index Cond: ((name)::text ~~ '%мяч%'::text)
                     ->  Bitmap Index Scan on trgm_idx_taginfo_name  (cost=0.00..220.97 rows=1196 width=0) (actual time=148.407..148.408 rows=1791 loops=1)
                           Index Cond: ((name)::text ~~ '%баскетбольный%'::text)
$ select name, frequency_30 from taginfo where name like '%платье%' or name like '%кофта%' order by frequency_30 limit 200;
Time: 24284.050 ms (00:24.284)
```
 
Время выполнения конечного тестового запроса в mongo: 0.5c, в postgresql: 24с

```sql

SELECT indexname, indexdef FROM pg_indexes WHERE tablename='taginfo';
drop index trgm_idx_taginfo_name;
CREATE INDEX idx ON taginfo USING GIN (to_tsvector('russian',name));

explain analyze select name, frequency_30 from taginfo where name like '%мяч%' or name like '%баскетбольный%' order by frequency_30 limit 200;
                                                                QUERY PLAN                                                                 
-------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=343820.57..343843.90 rows=200 width=61) (actual time=7660.514..7663.273 rows=200 loops=1)
   ->  Gather Merge  (cost=343820.57..344053.22 rows=1994 width=61) (actual time=7657.342..7660.082 rows=200 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=342820.55..342823.04 rows=997 width=61) (actual time=7638.717..7638.738 rows=200 loops=3)
               Sort Key: frequency_30
               Sort Method: top-N heapsort  Memory: 68kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 74kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 69kB
               ->  Parallel Seq Scan on taginfo  (cost=0.00..342777.46 rows=997 width=61) (actual time=10.338..7624.144 rows=6760 loops=3)
                     Filter: (((name)::text ~~ '%мяч%'::text) OR ((name)::text ~~ '%баскетбольный%'::text))
                     Rows Removed by Filter: 3980091
\timing
select name, frequency_30 from taginfo where name like '%платье%' or name like '%кофта%' order by frequency_30 limit 200;
Time: 7843.323 ms (00:07.843)
select name, frequency_30 from taginfo where name like '%мяч%' or name like '%баскетбольный%' order by frequency_30 limit 200;
Time: 7668.810 ms (00:07.669)
```
