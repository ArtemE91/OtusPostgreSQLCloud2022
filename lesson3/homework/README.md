## Домашнее задание 3 Занятие  
  
  
### Настройка PostgreSQL

* создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a или ЯО
* поставьте на нее PostgreSQL 14 через sudo apt
```shell
$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14 
```
* проверьте что кластер запущен
```shell
$ sudo -u postgres pg_lsclusters 
```
* зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```sql
create table test(c1 text);
insert into test values ('a'), ('b'), ('c'), ('d'), ('e');
```
* остановите postgres
```shell
$ sudo -u postgres pg_ctlcluster 14 main stop 
```
* создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB - или аналог в другом облаке/виртуализации
* добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
* проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
Установим утилиту `parted` чтобы разбить диск на разделы
```shell
$ sudo apt-get install parted 
```
Идентифицируем диск на сервере
```shell
$ sudo parted -l | grep Error
Error: `/dev/vdb`: unrecognised disk label
$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  15G  0 disk 
├─vda1 252:1    0   1M  0 part 
└─vda2 252:2    0  15G  0 part /
`vdb`   252:16   0  20G  0 disk 
```
 Указываю стандарт разметки GPT
```shell
$ sudo parted /dev/vdb mklabel gpt
$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  15G  0 disk 
├─vda1 252:1    0   1M  0 part 
└─vda2 252:2    0  15G  0 part /
`vdb`    252:16   0  20G  0 disk 
└─vdb1 252:17   0  20G  0 part
```
Создание файловой системы на новом разделе
```shell
$ sudo mkfs.ext4 -L datapartition /dev/vdb1
$ sudo lsblk --fs
NAME   FSTYPE LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                             
├─vda1                                                                          
└─vda2 ext4                 82afb880-9c95-44d6-8df9-84129f3f2cd1     11G    21% /
vdb                                                                             
└─vdb1 ext4   datapartition 0d97edda-4dcb-411d-b18e-b929f0c37f8a  
```
Подключение новой файловой системы
```shell
$ sudo mkdir -p /mnt/data
```
Автоматическое монтирование файловой системы при загрузке
```shell
$ sudo vim /etc/fstab
```
Добавил запись `LABEL=datapartition /mnt/data ext4 defaults 0 2`
* перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```shell
$ sudo lsblk --fs
NAME   FSTYPE LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                             
├─vda1                                                                          
└─vda2 ext4                 82afb880-9c95-44d6-8df9-84129f3f2cd1     11G    21% /
vdb                                                                             
└─vdb1 ext4   datapartition 0d97edda-4dcb-411d-b18e-b929f0c37f8a   18.5G     0% /mnt/data
```
* сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```shell
$ sudo chown -R postgres:postgres /mnt/data/ 
```
* перенесите содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data
```shell
$ sudo mv /var/lib/postgresql/14 /mnt/data 
```
* попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
```shell
$ sudo -u postgres pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist
```
* напишите получилось или нет и почему

 Не получилось так как pg_ctl пытается использовать данные из каталога /var/lib/postgresql/14/main
* задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его

 Параметр который необходимо изменить находится в `postgresql.conf` -> `data_directory`
* напишите что и почему поменяли

  Создадим в директории conf.d `change_dir.conf` и пропишем этот параметр `data_directory = '/mnt/data/14/main'`
* попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start
```shell
$ sudo -u postgres pg_ctlcluster 14 main start
```
* напишите получилось или нет и почему

 Кластер успешно запустился так как мы изменили каталог из которого нужно использовать данные
* зайдите через через psql и проверьте содержимое ранее созданной таблицы
 ```sql
postgres=# select * from test;
 c1 
----
 a
 b
 c
 d
 e
(5 rows)
```