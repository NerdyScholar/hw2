# Homework 3


Создаем ВМ с Ubuntu 20.04:

```
 nikolai@MacBook-Pro-Nikolaj ~ % yc compute instances show otus-vm
id: fhmm85ui9dmpdpaciqb6
folder_id: b1gcs9c6cga6tikud9bb
created_at: "2024-02-26T20:14:10Z"
name: otus-vm
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmkaak3soikdh1nc7tt
  auto_delete: true
  disk_id: fhmkaak3soikdh1nc7tt
network_interfaces:
  - index: "0"
    mac_address: d0:0d:16:41:7d:24
    subnet_id: e9bbb0l0f0avklichech
    primary_v4_address:
      address: 192.168.0.10
      one_to_one_nat:
        address: 84.201.174.188
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: INSTANCE_METADATA
gpu_settings: {}
fqdn: otus-vm.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}

nikolai@MacBook-Pro-Nikolaj ~ % yc compute instances list
+----------------------+---------+---------------+---------+----------------+--------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+---------+---------------+---------+----------------+--------------+
| fhmm85ui9dmpdpaciqb6 | otus-vm | ru-central1-a | RUNNING | 84.201.174.188 | 192.168.0.10 |
+----------------------+---------+---------------+---------+----------------+--------------+
```


Ставим  Docker Engine:

```
  yc-user@otus-vm:~$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
# Executing docker install script, commit: e5543d473431b782227f8908005543bb4389b8de

```



Создаем каталог /var/lib/postgres, создаем контейнер с PostgreSQL 15:

```
  yc-user@otus-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
e1caac4eb9d2: Pull complete 

```


Развернем контейнер с клиентом postgres, попробуем создать БД:

```
yc-user@otus-vm:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres: 
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# CREATE DATABASE otus; 
CREATE DATABASE

```



Вставим в первой сессии новую строку:

```
  postgres=# -- Создаем таблицу
CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data VARCHAR(100)
);

-- Заполняем таблицу 10 рандомными строками
DO $$
BEGIN
    FOR i IN 1..10 LOOP
        INSERT INTO test (data) VALUES (md5(random()::text));
    END LOOP;
END $$;
CREATE TABLE
DO

```

```
postgres=# select * from test;
 id |               data               
----+----------------------------------
  1 | 18f8dc304ab66977b9b252e1bf7768fc
  2 | fd2bbb3dc35e4eec941208319bc856c9
  3 | 634373190a61e5623bf4cb6340dbb183
  4 | a1b5e39a1c71ca7967f55d02c0d6cee1
  5 | b76d1b71837f2fa62b068460c83d365e
  6 | 9fdd53cdd1be1a78678260c33f4e82dc
  7 | 928d09dfed349c793bd0e4e43fea469f
  8 | e221fb798a766e982af675531e6a8d94
  9 | f23419d368c956dc0613114269b4cc69
 10 | 070c7b4c382d52ecdd2c6ba6fd614371
 11 | 8f5ac4e97a11e416ca4bf1c1d28f6e95
 12 | 9b5a3aecb48088fce25fec3ff64aaca7
 13 | e3ec5acda20a8836555f7ed064b6a365
 14 | bb3cc001e87f613c8785d5112b4c9f75
 15 | 8bd64f7737f996112292cc2c9edeec08
 16 | f9202f936279d5da5658c28a8885290a
 17 | ec2889315eb1c3227ad7d150e13cf0e1
 18 | 05b4f768815ec2d2734048a84c2f5d92
 19 | c24e2efd46df2c219323d74ff19a8cc2
 20 | b0ba712283f328ed45f2dbadbdb9ce3a
(20 rows)

```



Подключится к контейнеру с сервером с ноутбука:
```
nikolai@MacBook-Pro-Nikolaj ~ % psql -p 5432 -U postgres -h 84.201.174.188 -d postgres -W
Password: 
psql (14.11 (Homebrew), server 15.6 (Debian 15.6-1.pgdg120+2))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

```



Удалим контейнер с сервером и создадим его заново:

```
yc-user@otus-vm:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
c79f809d84bd   postgres:15   "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

yc-user@otus-vm:~$ sudo docker stop c79f809d84bd
c79f809d84bd

yc-user@otus-vm:~$ sudo docker rm c79f809d84bd
c79f809d84bd

yc-user@otus-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
7013de85d52c8bed42d826167c02a5e93525b3fb06949279c93cfe596e16cb0d

yc-user@otus-vm:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
7013de85d52c   postgres:15   "docker-entrypoint.s…"   50 seconds ago   Up 49 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```


Проверим, остались ли данные после пересоздания:

```
  yc-user@otus-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
7013de85d52c8bed42d826167c02a5e93525b3fb06949279c93cfe596e16cb0d

yc-user@otus-vm:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres: 
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# select * from test;
 id |               data               
----+----------------------------------
  1 | 18f8dc304ab66977b9b252e1bf7768fc
  2 | fd2bbb3dc35e4eec941208319bc856c9
  3 | 634373190a61e5623bf4cb6340dbb183
  4 | a1b5e39a1c71ca7967f55d02c0d6cee1
  5 | b76d1b71837f2fa62b068460c83d365e
  6 | 9fdd53cdd1be1a78678260c33f4e82dc
  7 | 928d09dfed349c793bd0e4e43fea469f
  8 | e221fb798a766e982af675531e6a8d94
  9 | f23419d368c956dc0613114269b4cc69
 10 | 070c7b4c382d52ecdd2c6ba6fd614371
 11 | 8f5ac4e97a11e416ca4bf1c1d28f6e95
 12 | 9b5a3aecb48088fce25fec3ff64aaca7
 13 | e3ec5acda20a8836555f7ed064b6a365
 14 | bb3cc001e87f613c8785d5112b4c9f75
 15 | 8bd64f7737f996112292cc2c9edeec08
 16 | f9202f936279d5da5658c28a8885290a
 17 | ec2889315eb1c3227ad7d150e13cf0e1
 18 | 05b4f768815ec2d2734048a84c2f5d92
 19 | c24e2efd46df2c219323d74ff19a8cc2
 20 | b0ba712283f328ed45f2dbadbdb9ce3a
(20 rows)

```



Данные на месте, все в порядке!


