## Домашнее задание 4 Занятие  
  
  
### Тюнинг postgres

### Развернуть Постгрес на ВМ
   
#### Параметры виртуальной машины:
1) DB Version: 14
2) OS Type: linux Ubuntu 20.04
3) DB Type: web
4) Total Memory (RAM): 2 GB
5) CPUs num: 2
6) Data Storage: ssd


### Протестировать pg_bench

Протестируем postgres на дефолтных настройках

```shell
$ pgbench -i test
$ pgbench -c 50 -j 2 -P 10 -T 60 test
```

| params                                    | default1 | default2 | default3 | default4 | default5 | average     |
|-------------------------------------------|----------|----------|----------|----------|----------|-------------|
| number of transactions actually processed | 27112    | 34862    | 27652    | 31088    | 33399    | 30822.6     |
| latency average                           | 110.595  | 86.031   | 108.613  | 96.508   | 89.782   | 98.3058 ms  |
| latency stddev                            | 160.238  | 109.712  | 158.850  | 126.304  | 122.130  | 135.4468 ms |
| initial connection time                   | 68.664   | 59.965   | 60.398   | 59.603   | 60.887   | 61.9034 ms  |
| tps                                       | 451.512  | 580.313  | 459.214  | 517.241  | 556.044  | 512.8653    |

Протестируем postgres на настройках сгенерированных через https://pgtune.leopard.in.ua
```shell
$ cd /etc/postgresql/14/main/conf.d
$ sudo vim pgbench.conf
$ sudo -i -u postgres
$ /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main/ restart
$ psql
$ postgres=# show shared_buffers;
 shared_buffers 
----------------
 512MB
$ postgres=# /q
$ pgbench -i test
$ pgbench -c 50 -j 2 -P 10 -T 60 test
```
Сгенерированный конфиг
```yaml
# DB Version: 14
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 2 GB
# CPUs num: 2
# Connections num: 100
# Data Storage: ssd

max_connections = 100
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 5242kB
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
```

| params                                    | leopard1 | leopard2 | leopard3 | leopard4 | leopard5 | average    |
|-------------------------------------------|----------|----------|----------|----------|----------|------------|
| number of transactions actually processed | 33506    | 38248    | 30241    | 35026    | 31767    | 33757.6    |
| latency average                           | 89.500   | 78.421   | 99.229   | 85.626   | 94.409   | 89.436 ms  |
| latency stddev                            | 116.463  | 97.810   | 134.834  | 108.364  | 129.115  | 117.317 ms |
| initial connection time                   | 67.125   | 64.089   | 65.239   | 62.814   | 62.105   | 64.27 ms   |
| tps                                       | 557.852  | 636.328  | 502.903  | 583.092  | 528.894  | 561.813    |

Протестируем postgres на настройках сгенерированных через http://pgconfigurator.cybertec.at/
```shell
$ exit
$ sudo vim pgbench.conf
$ sudo -i -u postgres
$ /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main/ restart
$ psql
$ postgres=# select sourcefile from pg_file_settings where name='work_mem';
                 sourcefile                  
---------------------------------------------
 /etc/postgresql/14/main/conf.d/pgbench.conf
$ postgres=# /q
$ pgbench -i test
$ pgbench -c 50 -j 2 -P 10 -T 60 test
```

Сгенерированный конфиг
```yaml
# Connectivity
max_connections = 100
superuser_reserved_connections = 3
 
# Memory Settings
shared_buffers = '512 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '1 GB'
effective_io_concurrency = 100   # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)
 
# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any
 
# Replication
wal_level = replica		# consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = on
 
# Checkpointing: 
checkpoint_timeout  = '15 min' 
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

 
# WAL writing
wal_compression = on
wal_buffers = -1    # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

 
# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0
 
# Parallel queries: 
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 2
parallel_leader_participation = on
 
# Advanced features 

enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
```

| params                                    | cybertec1 | cybertec2 | cybertec3 | cybertec4 | cybertec5 | average    |
|-------------------------------------------|-----------|-----------|-----------|-----------|-----------|------------|
| number of transactions actually processed | 37038     | 30951     | 30699     | 34406     | 36404     | 33899.6    |
| latency average                           | 80.968    | 96.946    | 97.681    | 87.167    | 82.370    | 89.0264 ms |
| latency stddev                            | 108.076   | 138.795   | 122.655   | 115.352   | 114.915   | 119.958 ms |
| initial connection time                   | 62.724    | 64.441    | 66.391    | 62.529    | 66.527    | 64.52 ms   |
| tps                                       | 616.610   | 514.665   | 511.101   | 572.788   | 606.056   | 564.244    |


### Настроить кластер на оптимальную производительность не обращая внимания на стабильность БД

Отключим синхронную запись на диск
```shell
$ sudo vim pgbench.conf
# fsync=off
# full_page_writes=off
$ sudo -i -u postgres
$ /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main/ restart
$ postgres=# /q
$ pgbench -i test
$ pgbench -c 50 -j 2 -P 10 -T 60 test
```

| params                                    | async1   | async2  | async3   | async4  | async5   | average   |
|-------------------------------------------|----------|---------|----------|---------|----------|-----------|
| number of transactions actually processed | 121807   | 118547  | 121878   | 117432  | 119606   | 119854    |
| latency average                           | 24.547   | 25.223  | 24.531   | 25.464  | 25.000   | 24.952 ms |
| latency stddev                            | 21.195   | 26.152  | 21.593   | 27.631  | 21.577   | 23.629 ms |
| initial connection time                   | 63.960   | 65.031  | 67.479   | 63.331  | 63.570   | 64.674 ms |
| tps                                       | 2028.806 | 1974.56 | 2030.072 | 1955.89 | 1992.134 | 1996.292  |

### Сводная таблица

| params                                    | default     | leopard    | cybertec   | async     |
|-------------------------------------------|-------------|------------|------------|-----------|
| number of transactions actually processed | 30822.6     | 33757.6    | 33899.6    | 119854    |
| latency average                           | 98.3058 ms  | 89.436 ms  | 89.0264 ms | 24.952 ms |
| latency stddev                            | 135.4468 ms | 117.317 ms | 119.958 ms | 23.629 ms |
| initial connection time                   | 61.9034 ms  | 64.27 ms   | 64.52 ms   | 64.674 ms |
| tps                                       | 512.8653    | 561.813    | 564.244    | 1996.292  |

Как видно из таблицы самой плохой результат при дефолтных настройках.