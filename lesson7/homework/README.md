## Домашнее задание 7 Занятие


### Postgresql в minikube

Устанавливаем minikube на систему
```shell
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
$ sudo install minikube-darwin-arm64 /usr/local/bin/minikube
$ minikube start
```

Создаю namespace и делаю чтоб он использовался по умолчанию
```shell
$ kubectl create namespace pg-kub1
namespace/pg-kub1 created
$ kubectl config set-context --current --namespace=pg-kub1
Context "minikube" modified.
```

Настроим переменное окружение для подключения к докеру внутри minikube
```shell
$ minikube docker-env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://127.0.0.1:55316" 
export DOCKER_CERT_PATH="/Users/arteme/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"
```

Соберем образ app и запускаем Pod
```shell
$ docker build -t pg_kub1_app:v1 .
$ kubectl apply -f pod.yml
pod/hello-demo created 
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
hello-demo   1/1     Running   0          36s
```

Проверим работу приложения
```shell 
$ minikube ssh
$ curl 172.17.0.5
Hello world from hello-demo ! v 2021-1
```

Пробросим порты
```shell
$ kubectl port-forward hello-demo 8000:80
-- переходим в браузер на http://localhost:8000
Hello world from hello-demo ! v 2021-1
```

Удалим Pod
```shell
$ kubectl delete -f pod.yml
pod "hello-demo" deleted
```

Посмотрим addon и добавим ingress
```shell
$ minikube addons list
-- необходимо добавить ingress
$ minikube addons enable ingress
```

Создадим deployment и service
```shell
$ kubectl apply -f deployment.yaml 
deployment.apps/hello-deployment created
$ kubectl apply -f service.yml    
service/hello-service created
$ kubectl apply -f ingress.yml
ingress.networking.k8s.io/hello-ingress created
$ minikube tunnel
-- перейдем по url http://localhost/myapp
Hello world from hello-deployment-6f75dd7f75-jp8xf ! v 2021-1
-- http://localhost/myapp/health
{"status": "ok"}
-- удалим все что делали
```

Развернеи StatefullSet
```shell
$ cd ../postgres
$ kubectl apply -f postgres.yml
-- statefulset.apps/postgres-statefulset created
$ kubectl port-forward postgres-statefulset-0 5433:5432
-- проверим подключение к бд
$ psql -h localhost -p 5433 -U myuser -W myapp
myapp=# \l
                              List of databases
   Name    | Owner  | Encoding |  Collate   |   Ctype    | Access privileges 
-----------+--------+----------+------------+------------+-------------------
 myapp     | myuser | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | myuser | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | myuser | UTF8     | en_US.utf8 | en_US.utf8 | =c/myuser        +
           |        |          |            |            | myuser=CTc/myuser
 template1 | myuser | UTF8     | en_US.utf8 | en_US.utf8 | =c/myuser        +
           | 
-- создадим таблицу
myapp=# create table client(id serial, name text);
CREATE TABLE
myapp=# insert into client(name) values ('Ivan'), ('Sergey'), ('Alexander');
INSERT 0 3
```

Соберем приложение
```shell
$ cd app
$ docker build -t pg_kub2_postgres:v1 .
$ cd ..
$ kubectl apply -f secrets.yml -f app-config.yml -f deployment.yml -f service.yml
$ kubectl port-forward pg-deployment-7847768fcc-d5gcw 8002:80
```

Переходим по адресу localhost:8002/db
Ответ
```shell
[{"name": "Ivan", "id": 1}, {"name": "Sergey", "id": 2}, {"name": "Alexander", "id": 3}]
```
