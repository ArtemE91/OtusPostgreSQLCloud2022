# Postgres & Docker


### Set up the repository

1) Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```shell
  $ sudo apt-get update
  $ sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
2) Add Docker’s official GPG key:
```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
3) Install Docker Engine
```shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Создаем сеть Docker
```shell
docker network create my-app-psql
```

Запускаем контейнер postgresql
```shell
docker run -d --name postgres --network my-app-psql -e POSTGRES_PASSWORD=111111 -v postgres:/var/lib/postgresql/data postgres:14
```

Клиент
```shell
docker run -d --name client --network my-app-psql -e POSTGRES_PASSWORD=111111 -v postgres:/var/lib/postgresql/data postgres:14
```

Подключаемся с клиента к серверу
```shell
psql -h postgresql_server -U courses
```

Создаем таблицу и наполняем данными
```sql
create table courses (id serial primary key, first_name varchar(50) not null, last_name varchar(50), course varchar(50) not null);

insert into courses (first_name, last_name, course) values ('Alex', 'Bolduin', 'Database'), ('Greg', 'Smitt', 'FrontEnd');
```