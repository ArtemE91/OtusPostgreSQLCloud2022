## Домашнее задание 5 Занятие  
  
  
### Бэкапы Постгреса

Делам бэкап Постгреса используя WAL-G или pg_probackup и восстанавливаемся на другом кластере

Устанавливаем PostgreSQL и WAL-G
```shell
$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
$ sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

$ wget https://github.com/wal-g/wal-g/releases/download/v1.1.1-rc/wal-g-pg-ubuntu-20.04-amd64.tar.gz && tar -zxvf wal-g-pg-ubuntu-20.04-amd64.tar.gz && sudo mv wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
$ sudo ls -l /usr/local/bin/wal-g
$ sudo rm -rf /home/backups && sudo mkdir /home/backups && sudo chmod 777 /home/backups
$ sudo su postgres
$ vim ~/.walg.json
```
Данные для бекапа будем хранить на примонтированном диске.

```json
{
    "WALG_FILE_PREFIX": "/mnt/data/backups",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/14/main",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}
```

Добавляем настройки в `postgresql.auto.conf`
```shell
echo "wal_level=replica" >> /var/lib/postgresql/14/main/postgresql.auto.conf
echo "archive_mode=on" >> /var/lib/postgresql/14/main/postgresql.auto.conf
echo "archive_command='wal-g wal-push \"%p\" >> /var/lib/postgresql/14/main/log/archive_command.log 2>&1' " >> /var/lib/postgresql/14/main/postgresql.auto.conf 
echo "archive_timeout=60" >> /var/lib/postgresql/14/main/postgresql.auto.conf 
echo "restore_command='wal-g wal-fetch \"%f\" \"%p\" >> /var/lib/postgresql/14/main/log/restore_command.log 2>&1' " >> /var/lib/postgresql/14/main/postgresql.auto.conf
```

Перезапускаем кластер PostgreSQL
```shell
pg_ctlcluster 14 main stop
pg_ctlcluster 14 main start
```

Создадим тестовую базу данных и заполним данными
```shell
psql otus -c "create table test(i int);"
psql otus -c "insert into test values (10), (20), (30);"
psql otus -c "select * from test;"
 i  
----
 10
 20
 30
(3 rows)
```

Пушим бекап указывая из какого кластера это делается
```shell
$ wal-g backup-push /var/lib/postgresql/14/main
$ wal-g backup-list
name                          modified             wal_segment_backup_start
base_000000010000000000000003 2022-05-21T13:59:21Z 000000010000000000000003
```

Сделаем update записи в БД и снова сделаем бекап
```shell
$ psql otus -c "UPDATE test SET i = 3 WHERE i = 30"
$ wal-g backup-push /var/lib/postgresql/14/main
$ wal-g backup-list
name                                                     modified             wal_segment_backup_start
base_000000010000000000000003                            2022-05-21T13:59:21Z 000000010000000000000003
base_000000010000000000000007_D_000000010000000000000003 2022-05-21T14:09:42Z 000000010000000000000007
```

Создадим новый кластер и восстановимся
```shell
$ pg_createcluster 14 main2
$ rm -rf /var/lib/postgresql/14/main2
$ wal-g backup-fetch /var/lib/postgresql/14/main2 LATEST
```

Создаем файл для восстановления из WAL и запустим кластер
```shell
$ touch "/var/lib/postgresql/14/main2/recovery.signal"
$ pg_ctlcluster 14 main2 start
```

Убедимся что все данные на месте
```shell
$ psql -p 5433 otus -c "select * from test;"
 i  
----
 10
 20
  3
(3 rows)
```
