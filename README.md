# Homework 7 (Логический уровень PostgreSQL)


Поднимаем ВМ и ставим на нее PostgreSQL, используя информаицию из уроков 3 и 6.
Проверим:
```
 yc-user@otus-vm:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

Создание новой базы данных testdb:

```
  yc-user@otus-vm:~$ sudo -u postgres psql
psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE


```


Заход в созданную базу данных под пользователем postgres:

```
yc-user@otus-vm:~$ psql -U postgres -d testdb
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "postgres"

```

Ошибку пофиксил, прописав пароль для пользователя postgres и впоследствии для пользователя testread, создаем схемы и роли, вставляем данные, выдаем права (делал без шпаргалки):


```
  postgres=# ALTER ROLE postgres WITH PASSWORD 'admin';
ALTER ROLE
postgres=# \q
yc-user@otus-vm:~$ psql -U postgres -h localhost -d testdb
Password for user postgres: 
psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
testdb=# CREATE TABLE testnm.t1 (c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
testdb=# CREATE ROLE read_only_role;
ERROR:  role "read_only_role" already exists
testdb=# GRANT CONNECT ON DATABASE testdb TO read_only_role;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO read_only_role;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO read_only_role;
GRANT
testdb=# CREATE USER testread WITH PASSWORD 'test123';
ERROR:  role "testread" already exists
testdb=# GRANT read_only_role TO testread;
NOTICE:  role "testread" is already a member of role "read_only_role"
GRANT ROLE
testdb=# \q

```
Попробуем зайти под пользователем testread и сделать селект:

```
 yc-user@otus-vm:~$ psql -U testread -h localhost -d testdb
Password for user testread: 
psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> SELECT * FROM testnm.t1;
 c1 
----
  1
(1 row)
         
```
Получилось успешно, так как в предыдущих шагах я выдал отдельно права на схему + создал таблицу с явным указанием схемы, иначе она бы создалась в public.


Далее попробуем создать таблицу и сделать инсерт из под роли testread:

```
  yc-user@otus-vm:~$ psql -U testread -h localhost -d testdb
Password for user testread: 
psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1

```
Это успешно получилось, так как мы создали таблицы в схеме public, в которой роль  public по умолчанию grant на все действия.
Чтобы это исправить, я подсмотрел в шпаргалку + попробовал действие 41 из ДЗ:


```
testdb=# REVOKE CREATE on SCHEMA public FROM public; 
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public; 
REVOKE
testdb=# \q
yc-user@otus-vm:~$ psql -U testread -h localhost -d testdb
Password for user testread: 
psql (14.11 (Ubuntu 14.11-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
INSERT 0 1

```
Как мы видим, создать таблицу у нас не получилось, но вставлять значения мы все еще можем, чтобы мы не могли из под testread инсертить нужно прописать следующее:


```
 REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM public;
REVOKE USAGE ON SCHEMA public FROM public;

```
После выполнения этих команд, пользователь testread не сможет выполнять операции над объектами в схеме public, включая вставку данных в таблицы.


