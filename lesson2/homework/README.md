## Домашнее задание 2 Занятие  
  
  
### Postgres & Docker  
  
 * сделать в ЯО инстанс с Ubuntu 20.04  
 * поставил на нем Docker Engine  по документации
```shell
$ sudo apt-get update  
$ sudo apt-get install \  
  ca-certificates \  curl \  gnupg \  lsb-release
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
 * развернул контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres  и контейнер с клиентом через compose файл
```yml
---  
  
version: '3.9'  
  
services:  
  postgresql_server:  
    image: "postgres:14.2"  
    container_name: server
    environment:  
      - POSTGRES_PASSWORD=111111  
      - POSTGRES_DB=courses  
      - POSTGRES_USER=courses  
    volumes:  
      - /var/lib/postgres:/var/lib/postgresql/data  
    ports:  
      - 5432:5432  
    healthcheck:  
      test: ["CMD-SHELL", "pg_isready"]  
      interval: 5s  
      timeout: 5s  
      retries: 5  
      start_period: 5s  
    networks:  
      my-app-psql:  
        aliases:  
          - postgresql_server  
  
  postgresql_client:  
    image: "postgres:14.2"
    container_name: client
    environment:  
      - POSTGRES_PASSWORD=111111  
    ports:  
      - 5433:5433  
    healthcheck:  
      test: ["CMD-SHELL", "pg_isready"]  
      interval: 5s  
      timeout: 5s  
      retries: 5  
      start_period: 5s  
    networks:  
      my-app-psql:  
        aliases:  
          - postgresql_client  
  
  
networks:  
  my-app-psql:
```

 * подключится из контейнера с клиентом к контейнеру с сервером и сделать  таблицу с парой строк 
 ```shell  
$ sudo docker ps -a  
CONTAINER ID   IMAGE           COMMAND                  CREATED             STATUS                    PORTS                                                 NAMES

708dec750cb3   postgres:14.2   "docker-entrypoint.s…"   About an hour ago   Up 58 seconds (healthy)   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp             server

fc5201f3fa90   postgres:14.2   "docker-entrypoint.s…"   About an hour ago   Up 42 seconds (healthy)   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   client

$ sudo docker exec -it fc5201f3fa90 bash
$ psql -h postgresql_server -U courses
```  
```sql
$ create table courses (id serial primary key, first_name varchar(50) not null, last_name varchar(50), course varchar(50) not null);

$ insert into courses (first_name, last_name, course) values ('Alex', 'Bolduin', 'Database'), ('Greg', 'Smitt', 'FrontEnd');

$ select * from courses;  
 id | first_name | last_name |  course  
  1 | Alex       | Bolduin   | Database 
  2 | Greg       | Smitt     | FrontEnd

```
 * подключился к контейнеру с сервером с ноутбука извне инстансов ЯО  
 ```shell
 $ psql "host=84.201.177.204 port=5432 dbname=courses user=courses   	target_session_attrs=read-write"
 ```
 * удалил контейнер с сервером  
 ```shell
 $ sudo docker stop 708dec750cb3
 $ sudo docker rm server

 $ sudo docker ps -a
 CONTAINER ID   IMAGE           COMMAND                  CREATED             STATUS                    PORTS                                                 NAMES

207469eeb15f   postgres:14.2   "docker-entrypoint.s…"   22 seconds ago      Up 13 seconds (healthy)   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp             server

fc5201f3fa90   postgres:14.2   "docker-entrypoint.s…"   About an hour ago   Up 16 minutes (healthy)   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   client
 ```
 * создал его заново  
```shell
$ sudo docker-compose -f docker-compose.yml create postgresql_server
$ sudo docker-compose -f docker-compose.yml start postgresql_server
```
 * подключился снова из контейнера с клиентом к контейнеру с сервером  
```shell
$ sudo docker exec -it fc5201f3fa90 bash
$ psql -h postgresql_server -U courses
```
 * проверить, что данные остались на месте  
```shell
select * from courses;

 id | first_name | last_name |  course  

----+------------+-----------+----------

  1 | Alex       | Bolduin   | Database

  2 | Greg       | Smitt     | FrontEnd
```