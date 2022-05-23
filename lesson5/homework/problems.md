### Проблемы


Устанавливаем PostgreSQL и WAL-G

```shell
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
sudo -u postgres pg_lsclusters 
sudo rm /usr/local/bin/wal-g
wget https://github.com/wal-g/wal-g/releases/download/v1.1.1-rc/wal-g-pg-ubuntu-20.04-amd64.tar.gz && tar -zxvf wal-g-pg-ubuntu-20.04-amd64.tar.gz && sudo mv wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
sudo ls -l /usr/local/bin/wal-g
sudo rm -rf /home/backups && sudo mkdir /home/backups && sudo chmod 777 /home/backups
```

Создаем конфиг для WAL-G
```json
{
    "WALG_FILE_PREFIX": "/home/backups",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/14/main",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}
```

Создаем папку для логов и добавляем настройки в postgresql.auto.conf
```shell
mkdir /var/lib/postgresql/14/main/log
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
cd /home/backups
```

Создадим БД и добавим данных
```shell
psql -c "CREATE DATABASE otus;"
psql otus -c "create table test(i int);"
psql otus -c "insert into test values (10), (20), (30);"
psql otus -c "select * from test;"
```

Делаем бекап
```shell
wal-g backup-push /var/lib/postgresql/14/main
wal-g backup-list
psql otus -c "UPDATE test SET i = 3 WHERE i = 30"
wal-g backup-push /var/lib/postgresql/14/main
wal-g backup-list
```

Восстановимся
```shell
pg_createcluster 14 main2
rm -rf /var/lib/postgresql/14/main2
wal-g backup-fetch /var/lib/postgresql/14/main2 LATEST
touch "/var/lib/postgresql/14/main2/recovery.signal"
pg_ctlcluster 14 main2 start
psql -p 5433 otus -c "select * from test;"
```
До этого момента все ОК Проходит. Дальше делаю следующее

```shell
pg_ctlcluster 14 main2 stop
pg_dropcluster --stop 14 main2
rm -rf /var/lib/postgresql/14/main2
psql otus -c "create table test(i int);"
psql otus -c "insert into test values (10), (20), (30);"
psql otus -c "select * from test;"
pg_createcluster 14 main2
rm -rf /var/lib/postgresql/14/main2
wal-g backup-fetch /var/lib/postgresql/14/main2 LATEST
touch "/var/lib/postgresql/14/main2/recovery.signal"
pg_ctlcluster 14 main2 start
```

Лог Ошибки
```shell
Error: /usr/lib/postgresql/14/bin/pg_ctl /usr/lib/postgresql/14/bin/pg_ctl start -D /var/lib/postgresql/14/main2 -l /var/log/postgresql/postgresql-14-main2.log -s -o  -c config_file="/etc/postgresql/14/main2/postgresql.conf"  exited with status 1: 
2022-05-23 17:03:17.628 UTC [11987] LOG:  starting PostgreSQL 14.3 (Ubuntu 14.3-1.pgdg20.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0, 64-bit
2022-05-23 17:03:17.628 UTC [11987] LOG:  listening on IPv6 address "::1", port 5433
2022-05-23 17:03:17.628 UTC [11987] LOG:  listening on IPv4 address "127.0.0.1", port 5433
2022-05-23 17:03:17.632 UTC [11987] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5433"
2022-05-23 17:03:17.639 UTC [11988] LOG:  database system was interrupted; last known up at 2022-05-23 17:01:57 UTC
2022-05-23 17:03:17.639 UTC [11988] LOG:  creating missing WAL directory "pg_wal/archive_status"
2022-05-23 17:03:17.686 UTC [11988] LOG:  restored log file "00000002.history" from archive
2022-05-23 17:03:17.718 UTC [11988] LOG:  starting archive recovery
2022-05-23 17:03:17.760 UTC [11988] LOG:  restored log file "00000002.history" from archive
2022-05-23 17:03:17.895 UTC [11988] LOG:  restored log file "000000010000000000000007" from archive
2022-05-23 17:03:18.819 UTC [11988] FATAL:  requested timeline 2 is not a child of this server's history
2022-05-23 17:03:18.819 UTC [11988] DETAIL:  Latest checkpoint is at 0/7000148 on timeline 1, but in the history of the requested timeline, the server forked off from that timeline at 0/6000000.
2022-05-23 17:03:18.821 UTC [11987] LOG:  startup process (PID 11988) exited with exit code 1
2022-05-23 17:03:18.821 UTC [11987] LOG:  aborting startup due to startup process failure
2022-05-23 17:03:18.822 UTC [11987] LOG:  database system is shut down
pg_ctl: could not start server
Examine the log output.
```

Попробуем поднять кластер main3
```shell
pg_createcluster 14 main3
rm -rf /var/lib/postgresql/14/main3
wal-g backup-fetch /var/lib/postgresql/14/main3 LATEST
touch "/var/lib/postgresql/14/main3/recovery.signal"
pg_ctlcluster 14 main3 start
```

Ошибки
```shell
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@14-main3
Error: /usr/lib/postgresql/14/bin/pg_ctl /usr/lib/postgresql/14/bin/pg_ctl start -D /var/lib/postgresql/14/main3 -l /var/log/postgresql/postgresql-14-main3.log -s -o  -c config_file="/etc/postgresql/14/main3/postgresql.conf"  exited with status 1: 
2022-05-23 18:43:15.508 UTC [12576] LOG:  starting PostgreSQL 14.3 (Ubuntu 14.3-1.pgdg20.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0, 64-bit
2022-05-23 18:43:15.509 UTC [12576] LOG:  listening on IPv6 address "::1", port 5434
2022-05-23 18:43:15.509 UTC [12576] LOG:  listening on IPv4 address "127.0.0.1", port 5434
2022-05-23 18:43:15.511 UTC [12576] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5434"
2022-05-23 18:43:15.517 UTC [12577] LOG:  database system was interrupted; last known up at 2022-05-23 17:01:57 UTC
2022-05-23 18:43:15.517 UTC [12577] LOG:  creating missing WAL directory "pg_wal/archive_status"
2022-05-23 18:43:15.564 UTC [12577] LOG:  restored log file "00000002.history" from archive
2022-05-23 18:43:15.596 UTC [12577] LOG:  starting archive recovery
2022-05-23 18:43:15.631 UTC [12577] LOG:  restored log file "00000002.history" from archive
2022-05-23 18:43:15.786 UTC [12577] LOG:  restored log file "000000010000000000000007" from archive
2022-05-23 18:43:16.682 UTC [12577] FATAL:  requested timeline 2 is not a child of this server's history
2022-05-23 18:43:16.682 UTC [12577] DETAIL:  Latest checkpoint is at 0/7000148 on timeline 1, but in the history of the requested timeline, the server forked off from that timeline at 0/6000000.
2022-05-23 18:43:16.684 UTC [12576] LOG:  startup process (PID 12577) exited with exit code 1
2022-05-23 18:43:16.684 UTC [12576] LOG:  aborting startup due to startup process failure
2022-05-23 18:43:16.685 UTC [12576] LOG:  database system is shut down
pg_ctl: could not start server
Examine the log output.
```