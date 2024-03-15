# Homework 9 (Журналы)



Настроим выполнение контрольной точки раз в 30 секунд. Для этого зайдем в файл postgresql.conf:
```
sudo nano /etc/postgresql/15/main/postgresql.conf

```

Изменим данный параметр в конфигурации

```
  checkpoint_timeout = 30s                # range 30s-1d
```
Теперь контрольные точки будут создаваться каждые 30 секунд.



Запустим pgbench с такими настройками на 10 минут:

```
  yc-user@otus-vm:~$ pgbench -c 10  -T 60 -U postgres -h localhost -d testdb

```
Результат:


```
  transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 230225
number of failed transactions: 0 (0.000%)
latency average = 26.059 ms
initial connection time = 96.686 ms
tps = 383.740628 (without initial connection time)
```
Посмотрим, какой объем журнальных файлов был сгенерирован за это время. Оценим, какой объем приходится в среднем на одну контрольную точку.

Вес лога до pgbench:

```
 yc-user@otus-vm:~$ cd /var/log/postgresql
yc-user@otus-vm:/var/log/postgresql$ du -h *.log
4K	postgresql-15-main.log 
```
Вес лога после pgbench:

``` yc-user@otus-vm:~$ cd /var/log/postgresql
yc-user@otus-vm:/var/log/postgresql$ du -h *.log
8K	postgresql-15-main.log 
```
Добавилось 4кБ журнала, теперь посмотрим, что по контрольным точкам:

Количество контрольных точек до pgbench:


```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# SELECT checkpoints_timed AS timed_checkpoints,
       checkpoints_req AS requested_checkpoints,
       checkpoint_write_time,
       checkpoint_sync_time
FROM pg_stat_bgwriter;
 timed_checkpoints | requested_checkpoints | checkpoint_write_time | checkpoint_sync_time 
-------------------+-----------------------+-----------------------+----------------------
                 6 |                     0 |                748583 |                   49
```

Количество контрольных точек после  pgbench:


```
 tps = 383.740628 (without initial connection time)

yc-user@otus-vm:~$ sudo -u postgres psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# SELECT checkpoints_timed AS timed_checkpoints,
       checkpoints_req AS requested_checkpoints,
       checkpoint_write_time,
       checkpoint_sync_time
FROM pg_stat_bgwriter;
 timed_checkpoints | requested_checkpoints | checkpoint_write_time | checkpoint_sync_time 
-------------------+-----------------------+-----------------------+----------------------
                28 |                     1 |               1404961 |                  385
```
Мы видим, что ене все контрольные точки создались по расписанию. Это связано с блокировками, которые могли быть во время попытки сделать новую контрольную точку.

Таким образом, примерно 0.2 кБ приходится на одну контрольную точку


Теперь давайте сравним tps в синхронном/асинхронном режиме утилитой pgbench:

Сначала запустим на 60 сек синхронный режим (то есть, режим по умолчанию):

```
yc-user@otus-vm:~$ pgbench -c 10 -T 60 -U postgres -h localhost  testdb
Password: 
pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 28352
number of failed transactions: 0 (0.000%)
latency average = 21.150 ms
initial connection time = 106.168 ms
tps = 472.823852 (without initial connection time)

```

Теперь зайдем в конфиг PostgreSQL и изменим параметр:

```
synchronous_commit = off 

```
Перезапустим кластер и проведем pgbench с теми же параметрами

Результат:

```
yc-user@otus-vm:~$ pgbench -c 10 -T 60 -U postgres -h localhost  testdb
Password: 
pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 98257
number of failed transactions: 0 (0.000%)
latency average = 6.098 ms
initial connection time = 99.832 ms
tps = 1639.788703 (without initial connection time)


```

Как мы видим, производительность выросла более чем в 3 раза. Это объясняется тем, что при асинхронной записи сброс журнальных записей выполняет процесс wal writer, чередуя циклы работы с ожиданием (которое устанавливается параметром wal_writer_delay = 200ms по умолчанию). Этот алгоритм нацелен на то, чтобы по возможности не синхронизировать одну и ту же страницу несколько раз, что важно при большом потоке изменений.

Асинхронная запись эффективнее синхронной — фиксация изменений не ждет записи.


Теперь создадим новый кластер с включенной контрольной суммой страниц через скрипт:


```
  #!/bin/bash

# Переменные с настройками
PG_VERSION=15           # Версия PostgreSQL
PG_CLUSTER_NAME=mydb    # Имя кластера
PG_DATA_DIR=/var/lib/postgresql/${PG_VERSION}/${PG_CLUSTER_NAME} # Директория данных кластера

# Установка PostgreSQL
sudo apt-get update
sudo apt-get install -y postgresql-${PG_VERSION}

# Создание кластера с включенными контрольными суммами
sudo pg_createcluster --data=${PG_DATA_DIR} ${PG_VERSION} ${PG_CLUSTER_NAME} -- --data-checksums

# Перезапуск сервера PostgreSQL
sudo systemctl restart postgresql@${PG_VERSION}-${PG_CLUSTER_NAME}

# Отображение статуса контрольных сумм
sudo -u postgres psql -c "SHOW data_checksums;"

```

Проверим, что у нас оба кластера живые:

```
yc-user@otus-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
15  mydb    5433 online postgres /var/lib/postgresql/15/mydb /var/log/postgresql/postgresql-15-mydb.log

```
Подключимся к новому кластеру и посмотрим, включены ли контрольные суммы:

```
yc-user@otus-vm:~$ sudo -u postgres psql -p 5433
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

postgres=# show data_checksums;
 data_checksums 
----------------
 on
(1 row)


```
Создадим таблицу для тестов, посмотрим, что в ней лежит:

```
postgres=# CREATE TABLE my_table (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     data TEXT
postgres(# );
CREATE TABLE
postgres=# INSERT INTO my_table (data) 
postgres-# SELECT md5(random()::text) || chr(trunc(65 + random() * 26)::int)
postgres-# FROM generate_series(1, 5);
INSERT 0 5
postgres=# SELECT * FROM my_table;
 id |               data                
----+-----------------------------------
  1 | 12970382bbd405114c91dad9c01ef6deG
  2 | f2e43a2086f7a7b01102a25d519254d7N
  3 | cb78b8761c0738e166a4065b2d5e93deV
  4 | 153e8515d840f451ef80991005d5cc41Z
  5 | 398dda7f50898e274771cb518bb12644P
(5 rows)


```
Посмотрим, где лежит эта таблица:

```
postgres=# SELECT pg_relation_filepath('my_table');
 pg_relation_filepath 
----------------------
 base/5/16389
(1 row)


```
Удалим оттуда пару байт:


```
yc-user@otus-vm:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/main/base/5/16389 oflag=dsync conv=notrunc bs=1 count=8
10+0 records in
10+0 records out
10 bytes copied, 0.0208113 s, 0.4 kB/s

```

Перезапустим кластер, попробуем заселектить нашу таблицу:

```
postgres=# SELECT * FROM my_table;
WARNING:  page verification failed, calculated checksum 134265 but expected 149530
ERROR:  invalid page in block 0 of relation  base/5/16389

```

Получаем наблюдаемую ошибку. Ппоробуем ее обойти:



```
postgres=# SET ignore_checksum_failure = on;
SET
postgres=# SELECT * FROM wallevel;
WARNING:  page verification failed, calculated checksum 134265 but expected 149530
 id |               data                
----+-----------------------------------
  1 | 12970382bbd405114c91dad9c01ef6deG
  2 | f2e43a2086f7a7b01102a25d519254d7N
  3 | cb78b8761c0738e166a4065b2d5e93deV
  4 | 153e8515d840f451ef80991005d5cc41Z
  5 | 398dda7f50898e274771cb518bb12644P

```
Данные прочитались, так как мы тронули только заголовки. С помошью прописанного параметра нам удалось прочитать таблицу. При этом мы берем во внимание, что данные без схождения контрольных сумм могут быть повреждены и некерректны.

