# Домашнее задание №2
Установка и настройка PostgteSQL в контейнере Docker

**Цель:**
* развернуть ВМ в GCP/ЯО/Аналоги
* установить туда докер
* установить PostgreSQL в Docker контейнере
* настроить контейнер для внешнего подключения
## Установка докера на виртуальную машину
```
root@otus-homework-2:~# apt install docker.io
```
## Создание каталога /var/lib/postgres
```
mkdir /var/lib/postgres
```
## Развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres
Так как не указал версию (postgres:14), установился контейнер с версией latest (16.3)
```
root@otus-homework-2:~# docker run -p 5432:5432 -e POSTGRES_PASSWORD=postgres -v /var/lib/postgres:/var/lib/postgresql/data -d --name otus_postgres postgres
Unable to find image 'postgres:latest' locally
latest: Pulling from library/postgres
b0a0cf830b12: Pull complete
2f87c12e2712: Pull complete
330a1c6ebd55: Pull complete
342b3a64b734: Pull complete
0a5b03808495: Pull complete
7fd25b6b5f08: Pull complete
bfe7c5c4e03d: Pull complete
edd49a43d68d: Pull complete
b513833c2b0c: Pull complete
c15cd24d5f84: Pull complete
2168a771a1aa: Pull complete
1f1c83f91f17: Pull complete
efb5bb986f57: Pull complete
5936f8dc17df: Pull complete
Digest: sha256:ba727f758a75cdd503c6b63db66a5fbc22ded0a228952e9d88e601621ad4de64
Status: Downloaded newer image for postgres:latest
394d982793ecb800d3ed95630e5b13baab53a734ea4dd8311d73ea0d21812ad2
root@otus-homework-2:~# docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS         PORTS                                       NAMES
394d982793ec   postgres   "docker-entrypoint.s…"   12 seconds ago   Up 8 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   otus_postgres
```
## Развернуть контейнер с клиентом postgres
```
docker run -it --name pg_client --link otus_postgres:postgres postgres  psql -h postgres -U postgres
```
## Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
```
postgres=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
(0 rows)
postgres=# insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
postgres=# insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
## Подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/Аналоги
Подключение с другого пк под OL7:
```
[root@localhost ~]# psql -h 158.160.150.160 -p 5432 -U postgres
Password for user postgres:
psql (13.3, server 16.3 (Debian 16.3-1.pgdg120+1))
WARNING: psql major version 13, server major version 16.
         Some psql features might not work.
Type "help" for help.

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
## Удалить контейнер с сервером
```
root@otus-homework-2:~# docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                       NAMES
394d982793ec   postgres   "docker-entrypoint.s…"   19 minutes ago   Up 19 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   otus_postgres
root@otus-homework-2:~# docker rm otus_postgres
Error response from daemon: You cannot remove a running container 394d982793ecb800d3ed95630e5b13baab53a734ea4dd8311d73ea0d21812ad2. Stop the container before attempting removal or force remove
root@otus-homework-2:~# docker stop otus_postgres
otus_postgres
root@otus-homework-2:~# docker rm otus_postgres
otus_postgres
root@otus-homework-2:~# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
## Создать его заново
```
root@otus-homework-2:~# docker run -p 5432:5432 -e POSTGRES_PASSWORD=postgres -v /var/lib/postgres:/var/lib/postgresql/data -d --name otus_postgres postgres
06e9753e4ff74d505f38ba483e779a1d0f98fba1e525a49adbd451816cf35c23
root@otus-homework-2:~# docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                       NAMES
06e9753e4ff7   postgres   "docker-entrypoint.s…"   4 seconds ago   Up 3 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   otus_postgres
```
## Подключится снова из контейнера с клиентом к контейнеру с сервером
```
root@otus-homework-2:~# docker run -it --name pg_client --link otus_postgres:postgres postgres  psql -h postgres -U postgres
docker: Error response from daemon: Conflict. The container name "/pg_client" is already in use by container "b32dee0378e32f7ec213e31b2dab27bae8d42c7eff37198d3349086149c4bd0e". You have to remove (or rename) that container to be able to reuse that name.
See 'docker run --help'.
root@otus-homework-2:~# docker rm pg_client
pg_client
root@otus-homework-2:~# docker run -it --name pg_client --link otus_postgres:postgres postgres  psql -h postgres -U postgres
Password for user postgres:
psql (16.3 (Debian 16.3-1.pgdg120+1))
Type "help" for help.
```
## Проверить, что данные остались на месте
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
