# Homework 8 (MVCC и вакуум)


Создадим сети, подсети и саму вм следующими командами:

```
nikolai@MacBook-Pro-Nikolaj ~ % yc vpc network create --name otus-net --description "otus-net" && \
yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet" && \
yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key ~/.ssh/id_rsa.pub && vm_ip_address=$(yc compute instance show --name otus-vm | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

```

Поднимем PostgreSQL:

```
yc-user@otus-vm:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```


создадим БД для тестов:

```
  yc-user@otus-vm:~$ sudo -u postgres psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

postgres=# create database testdb;
CREATE DATABASE

```

Запустим pgbench -i postgres:



```
  yc-user@otus-vm:~$ pgbench -i -U postgres -h localhost  -d postgres
```
Результат:

```
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.21 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.89 s, vacuum 0.04 s, primary keys 0.27 s).          
```

Запустим pgbench -c8 -P 6 -T 60 -U postgres postgres:
```
 yc-user@otus-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres -h localhost  -d postgres

```

Результат:


```
 scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 19321
number of failed transactions: 0 (0.000%)
latency average = 24.678 ms
latency stddev = 18.540 ms
initial connection time = 87.519 ms
tps = 322.312679 (without initial connection time)
```

Применим параметры настройки PostgreSQL из прикрепленного к материалам занятия файла


```
yc-user@otus-vm:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
```
Внесем параметры из файла:

```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

```
Перезапустим PostgreSQL для применения новых настроек:
```
yc-user@otus-vm:~$ sudo systemctl restart postgresql
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

```
Протестируем:

```
yc-user@otus-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres -h localhost  -d postgres
```

Результат:


```
  transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 20514
number of failed transactions: 0 (0.000%)
latency average = 23.227 ms
latency stddev = 16.633 ms
initial connection time = 92.901 ms
tps = 342.223652 (without initial connection time)
```
Видим, что увеличилась производительность и уменьшилась задержка из-за увеличения различной памяти


Создадим таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк:
```
yc-user@otus-vm:~$ sudo -u postgres psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE TABLE my_table (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     text_column TEXT
postgres(# );
CREATE TABLE
postgres=# INSERT INTO my_table (text_column)
postgres-# SELECT md5(random()::text)
postgres-# FROM generate_series(1, 1000000);
INSERT 0 1000000

```
Посмотрим размер файла с таблицей:

```
postgres=# SELECT pg_size_pretty(pg_relation_size('my_table'));
 pg_size_pretty 
----------------
 65 MB
(1 row)
```

Обновим 5 раз все строки:
```
postgres=# UPDATE my_table
postgres-# SET text_column = text_column || 'X';
UPDATE 1000000
postgres=# UPDATE my_table
SET text_column = text_column || 'X';
UPDATE 1000000
postgres=# UPDATE my_table
SET text_column = text_column || 'Q';
UPDATE 1000000
postgres=# UPDATE my_table
SET text_column = text_column || 'P';
UPDATE 1000000
postgres=# UPDATE my_table
SET text_column = text_column || 'O';
UPDATE 1000000
```
Дополнительно поэкспериментировал с удалением-вставкой:

```

postgres=# -- Добавление случайного количества строк
postgres=# DO $$
postgres$# DECLARE
postgres$#     num_inserts INT := floor(random() * 1000000) + 1; -- Генерация случайного количества строк от 1 до 1 млн
postgres$#     i INT := 1;
postgres$# BEGIN
postgres$#     WHILE i <= num_inserts LOOP
postgres$#         INSERT INTO my_table (text_column)
postgres$#         VALUES (md5(random()::text) || chr(trunc(65 + random() * 26)::int)); -- Генерация случайных данных для столбца text_column
postgres$#         i := i + 1;
postgres$#     END LOOP;
postgres$# END $$;
DO
postgres=# 
postgres=# -- Удаление случайного количества строк
postgres=# DELETE FROM my_table
postgres-# WHERE ctid IN (
postgres(#     SELECT ctid
postgres(#     FROM my_table
postgres(#     ORDER BY random()
postgres(#     LIMIT floor(random() * 500000) + 1 -- Генерация случайного количества строк от 1 до 500 тыс
postgres(# );
DELETE 498619

```
Посмотрим время последнего автовакуума после многих апдейтов и по истечении нескольких минут:

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs;
     relname      | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
------------------+------------+------------+--------+-------------------------------
 pgbench_history  |      20514 |          0 |      0 | 2024-03-13 19:51:45.283635+00
 pgbench_tellers  |         10 |          0 |      0 | 2024-03-13 19:51:45.28017+00
 my_table         |     600458 |          0 |      0 | 2024-03-13 20:38:49.54437+00
 pgbench_branches |          1 |          0 |      0 | 2024-03-13 19:51:45.276129+00
 pgbench_accounts |     100000 |       3757 |      3 | 2024-03-13 19:29:49.467218+00


```
Посмотреть размер файла с таблицей:


```
postgres=# SELECT pg_size_pretty(pg_relation_size('my_table'));
 pg_size_pretty 
----------------
 195 MB
(1 row)
```

отклюмим автовакуум, 10 раз обновим все строчки или добавим к каждой строчке любой символ:


```
postgres=# ALTER TABLE my_table SET (autovacuum_enabled = off);
ALTER TABLE
postgres=# SHOW autovacuum;
 autovacuum 
------------
 on
(1 row)

postgres=# delete from my_table where text_column is not null;
DELETE 600458
postgres=# INSERT INTO my_table (text_column) 
SELECT md5(random()::text)
FROM generate_series(1, 1000000);
INSERT 0 1000000
postgres=# UPDATE my_table                                      
SET                                                                           
    text_column = md5(random()::text); -- Генерация случайной строки MD5
UPDATE 1000000
postgres=# UPDATE my_table
SET 
    text_column = md5(random()::text); -- Генерация случайной строки MD5
UPDATE 1000000
postgres=# UPDATE my_table
SET 
    text_column = md5(random()::text); -- Генерация случайной строки MD5
UPDATE 1000000
postgres=# UPDATE my_table
SET 
    text_column = md5(random()::text); -- Генерация случайной строки MD5
UPDATE 1000000
```

Посмотрим теперь размер таблицы:


```
postgres=# SELECT pg_size_pretty(pg_relation_size('my_table'));
 pg_size_pretty 
----------------
 488 MB
(1 row)
```


Таблица выросла в более чем 2 раза, хотя мертвых строк нет. Автовакуум не высвобождает место на диске после очистки.
