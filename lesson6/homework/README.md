## Домашнее задание 6 Занятие  
  
  
### Кластер Patroni

Создаем 3 ВМ для etcd + 3 ВМ для Patroni +1 HA proxy (при проблемах можно на 1 хосте развернуть)
Инициализируем кластер
Проверяем отказоустойсивость
*настраиваем бэкапы через wal-g или pg_probackup

Разворачиваем 3 виртуальных машины и устанавливаем etcd
```shell
$ sudo apt update && sudo apt upgrade -y && sudo apt install -y etcd
```

Проверим что с etcd
```shell
$ ps -aef | grep etcd | grep -v grep
etcd        8209       1  0 16:39 ?        00:00:00 /usr/bin/etcd
etcd        8174       1  0 16:39 ?        00:00:00 /usr/bin/etcd
etcd        8219       1  0 16:39 ?        00:00:00 /usr/bin/etcd
```

Остановим etcd
```shell
$ sudo systemctl stop etcd

cat > temp.cfg << EOF 
ETCD_NAME="etcd1"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd1.ru-central1.internal:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd1.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1.ru-central1.internal:2380,etcd2=http://etcd2.ru-central1.internal:2380,etcd3=http://etcd3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
EOF
cat temp.cfg | sudo tee -a /etc/default/etcd

cat > temp.cfg << EOF 
ETCD_NAME="etcd2"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd2.ru-central1.internal:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd2.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1.ru-central1.internal:2380,etcd2=http://etcd2.ru-central1.internal:2380,etcd3=http://etcd3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
EOF
cat temp.cfg | sudo tee -a /etc/default/etcd

cat > temp.cfg << EOF 
ETCD_NAME="etcd3"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd3.ru-central1.internal:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1.ru-central1.internal:2380,etcd2=http://etcd2.ru-central1.internal:2380,etcd3=http://etcd3.ru-central1.internal:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
EOF
cat temp.cfg | sudo tee -a /etc/default/etcd
```

Запускаем etcd. Проверка автозагрузки. Проверка etcd-кластера
```shell
$ sudo systemctl start etcd
$ systemctl is-enabled etcd
enabled
$ etcdctl cluster-health
member 61f991406239b07 is healthy: got healthy result from http://etcd1.ru-central1.internal:2379
member 3376d9633be63daf is healthy: got healthy result from http://etcd3.ru-central1.internal:2379
member 64b82de471857189 is healthy: got healthy result from http://etcd2.ru-central1.internal:2379
cluster is healthy
```

Развернем 3 ВМ для Postgres и установим его на них
```shell
$ sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14
$ pg_lsclusters
```

Проверим доступность нодов с etcd и с pgsql
```shell
$ ping etcd1.ru-central1.internal
$ ping pgsql1.ru-central1.internal
```

### Установка Patroni. 

Ставим python на 1 ноде pgsql1
```shell
$ sudo apt-get install -y python3 python3-pip git mc
$ sudo pip3 install psycopg2-binary 
```

После установки ПО останавливаем и удаляем экземлпяр постгреса который запускается по-умолчанию
```shell
$ sudo -u postgres pg_ctlcluster 14 main stop
$ sudo -u postgres pg_dropcluster 14 main
```

Устанавливаем patroni
```shell
$ sudo pip3 install patroni[etcd]
```

Делаем симлинк
```shell
$ sudo ln -s /usr/local/bin/patroni /bin/patroni
```

Стартуем сервис
```shell
$ sudo vim /etc/systemd/system/patroni.service
```
Заполняем patroni.service
```ini
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
```

Конфигурируем patroni и запускаем
```shell
$ sudo vim /etc/patroni.yml
$ sudo systemctl is-enabled patroni 
disabled
$ sudo systemctl enable patroni 
Created symlink /etc/systemd/system/multi-user.target.wants/patroni.service → /etc/systemd/system/patroni.service.
$ sudo systemctl start patroni 
$ sudo patronictl -c /etc/patroni.yml list 
```

Устанавливаем patroni на 2 и 3 ноду аналогично 1
```shell
$ sudo patronictl -c /etc/patroni.yml list
+--------+-------------+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102480809882482579) --+----+-----------+
| pgsql1 | 10.129.0.25 | Leader  | running |  1 |           |
| pgsql2 | 10.129.0.8  | Replica | running |  1 |        16 |
| pgsql3 | 10.129.0.13 | Replica | running |  1 |         0 |
+--------+-------------+---------+---------+----+-----------+
```

Установим pg_bouncer на каждом хосте с patroni
```shell
$ sudo apt install -y pgbouncer
```

Создаем конфиг
```shell
cat > temp3.cfg << EOF 
[databases]
otus = host=127.0.0.1 port=5432 dbname=otus 
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = postgres
EOF
cat temp3.cfg | sudo tee -a /etc/pgbouncer/pgbouncer.ini
```

Скопируем пароль пользователя postgres
```shell
postgres=# create user admindb with password 'root123';
postgres=# select usename,passwd from pg_shadow;
postgres    | SCRAM-SHA-256$4096:LnRg8KZB0rRne2N+Yr6f1g==$p5b0fXBkkoUJojZVY56MffEeNzJDHmnKCGzXjK/YsmY=:7hui1woOmIEZb68npKivzU+9EeA/+4sHbTjn2M0iRz8=
```
Создаем /etc/pgbouncer/userlist.txt и добавляем пользователя с паролем
```text
"postgres" "SCRAM-SHA-256$4096:LnRg8KZB0rRne2N+Yr6f1g==$p5b0fXBkkoUJojZVY56MffEeNzJDHmnKCGzXjK/YsmY=:7hui1woOmIEZb68npKivzU+9EeA/+4sHbTjn2M0iRz8="
```

Установим net-tools
```shell
$ sudo apt install net-tools
$ netstat -pltn
```

Так как pgbouncer запустился как демон убьем его и запустим как сервис
```shell
$ sudo su postgres
$ ps -xf
$ kill <PID>
$ sudo -u postgres pgbouncer /etc/pgbouncer/pgbouncer.ini
```

Проверим подключение к БД
```shell
# sudo -u postgres psql -p 6432 -h 127.0.0.1 otus
```

Добавим в сервис команду для рестарта pgbouncer
```shell
$ sudo vim /lib/systemd/system/pgbouncer.service
[Service]
Restart=always
```

Нагрузим pgbench с другого хоста
```shell
$ sudo -u postgres pgbench -p 6432 -i -d otus -h 10.129.0.25
$ pgbench -p 6432 -c 20 -C -T 60 -P 1 -d otus -h 10.129.0.25
```

Просмотр статистики в pgbouncer
```sql
show servers;
SHOW STATS_TOTALS;
 database  | xact_count | query_count | bytes_received | bytes_sent | xact_time | query_time | wait_time 
-----------+------------+-------------+----------------+------------+-----------+------------+-----------
 otus      |       6753 |       47128 |        4437029 |    1866838 | 948511868 |  884081326 |     44684
 pgbouncer |          1 |           1 |              0 |          0 |         0 |          0 |         0
(2 rows
show pools;
 database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us | pool_mode 
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-----------
 otus      | postgres  |        19 |          0 |             0 |        19 |       1 |       0 |         0 |        0 |       0 |          0 | session
 pgbouncer | pgbouncer |         1 |          0 |             0 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | statement
-- Поставить на паузу коннекты:
pause otus;
-- Возобновить коннект:
resume otus;
```

Разворачиваем 1 ВМ c HAProxy
```shell
$ sudo apt install -y --no-install-recommends software-properties-common && sudo add-apt-repository -y ppa:vbernat/haproxy-2.5 && sudo apt install -y haproxy=2.5.\*
```

Посмотрим как нам отвечают ноды
```shell
$ curl -v 10.129.0.25:8008/master
HTTP/1.0 200 OK
$ curl -v 10.129.0.8:8008/replica
HTTP/1.0 200 OK
$ curl -v 10.129.0.13:8008/master
HTTP/1.0 503 Service Unavailable
```

Протестируем доступ. Для этого установим клиент postgresql
```shell
$ sudo apt update && sudo apt upgrade -y && sudo apt install -y postgresql-client-common && sudo apt install postgresql-client -y
$ psql -p 6432 -d otus -h 10.129.0.25 -U postgres
```

Настраиваем конфиг HAProxy
```shell
$ sudo vim /etc/haproxy/haproxy.cfg
```

Добавляем в конфиг
```text
listen postgres_write
    bind *:5432
    mode            tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /master
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql1 10.129.0.25:6432 check port 8008
    server pgsql2 10.129.0.8:6432 check port 8008
    server pgsql3 10.129.0.13:6432 check port 8008

listen postgres_read
    bind *:5433
    mode            tcp
    http-check connect
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql1 10.129.0.25:6432 check port 8008
    server pgsql2 10.129.0.8:6432 check port 8008
    server pgsql3 10.129.0.13:6432 check port 8008
```

Рестартуем сервис
```shell
$ sudo systemctl restart haproxy.service
$ sudo systemctl status haproxy.service
```

Подключимся
```shell
$ psql -h localhost -d otus -U postgres -p 5432
$ \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 haproxy   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
```

Остановим мастер и проверим что все переключится:
```shell
$ sudo systemctl stop patroni 
etemik@pgsql1:~$ sudo patronictl -c /etc/patroni.yml list
+--------+-------------+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102480809882482579) --+----+-----------+
| pgsql1 | 10.129.0.25 | Replica | stopped |    |   unknown |
| pgsql2 | 10.129.0.8  | Replica | running |  2 |         0 |
| pgsql3 | 10.129.0.13 | Leader  | running |  2 |           |
+--------+-------------+---------+---------+----+-----------+
```

Как видим Leader переключился на 3 ноду. Запустим 1 ноду
```shell
$ sudo systemctl start patroni
+--------+-------------+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102480809882482579) --+----+-----------+
| pgsql1 | 10.129.0.25 | Replica | running |  2 |         0 |
| pgsql2 | 10.129.0.8  | Replica | running |  2 |         0 |
| pgsql3 | 10.129.0.13 | Leader  | running |  2 |           |
+--------+-------------+---------+---------+----+-----------+
```

Первая нода запустилась и донакатилась до Leader. Убьем 1 etcd и повторим тоже с нодами postgres
```shell
$ sudo systemctl stop etcd
$ sudo patronictl -c /etc/patroni.yml list
+--------+-------------+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102480809882482579) --+----+-----------+
| pgsql1 | 10.129.0.25 | Replica | running |  3 |         0 |
| pgsql2 | 10.129.0.8  | Leader  | running |  3 |           |
+--------+-------------+---------+---------+----+-----------+
-- 1 нода etcd
$ sudo systemctl stop etcd
-- 3 нода psql
$ sudo systemctl start patroni
 sudo patronictl -c /etc/patroni.yml list
+--------+-------------+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+ Cluster: patroni (7102480809882482579) --+----+-----------+
| pgsql1 | 10.129.0.25 | Replica | running |  3 |         0 |
| pgsql2 | 10.129.0.8  | Leader  | running |  3 |           |
| pgsql3 | 10.129.0.13 | Replica | running |  3 |         0 |
+--------+-------------+---------+---------+----+-----------+
```
Все поднялось