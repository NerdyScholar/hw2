# Homework 10 (Блокировки)


Поднимаем ВМ и ставим на нее PostgreSQL, используя информацию из уроков 3 и 6.
Проверим:
```
 yc-user@otus-vm:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

Посмотрим через nano файл postgresql.conf и внесем такие изменения:

```
log_lock_waits = on
deadlock_timeout = 200ms


```
Это позволит логгировать блокировки как требует условие.
Далее перезапустим кластер и воспроизведем ситуацию, при которой блокировка попадет в лог:

Создаем таблицу:
```
postgres=# CREATE TABLE my_table (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     data TEXT
postgres(# );
CREATE TABLE

```
Наполняем ее данными:



```
 postgres=# INSERT INTO my_table (data) 
SELECT md5(random()::text) || chr(trunc(65 + random() * 26)::int)
FROM generate_series(1, 10);
INSERT 0 10


```

Начнем первую транзакцию:
```
postgres=# begin;
BEGIN
postgres=*# SELECT * FROM my_table  FOR UPDATE;
  id   |               data                
-------+-----------------------------------
 50001 | c7b28817ea0a9e92d20c8a681b007661F
 50002 | 763f64c9604a7968e7ac759c0a3d992dH
 50003 | 6ff02989016f5195c36c4e569529683eI
 50004 | 313667c75df6bb2cb858781ab6e9e639M
 50005 | 4abdd8fbf645a6c6d1db7f26d4dd3a03S
 50006 | e059f98e36dff9d1e871712710f045ddW
 50007 | 0b5d337f0bca7bc65e723ed9ff6e2c46X
 50008 | 5ef464abe0f3cd04c004c3bdac40a3f3Y
 50009 | a217b5431e5251c029e4e71c4d202cd9J
 50010 | 28bafcce40fdc6702a2ecf11919cc141B
(10 rows)
      
```
Начнем вторую транзакцию:

```
postgres=# UPDATE my_table  SET data = 'new_data' WHERE id = 50001;
```
Она не завершается, после коммита в первой сессии завершилась и эта транзакция.

Посмотрим логи:


```
 2024-03-17 17:14:52.668 UTC [6997] postgres@postgres LOG:  process 6997 still waiting for ShareLock on transaction 1067 after 200.186 ms
2024-03-17 17:14:52.668 UTC [6997] postgres@postgres DETAIL:  Process holding the lock: 6925. Wait queue: 6997.
2024-03-17 17:14:52.668 UTC [6997] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "my_table"
2024-03-17 17:14:52.668 UTC [6997] postgres@postgres STATEMENT:  UPDATE my_table  SET data = 'new_data' WHERE id = 50001;
2024-03-17 17:16:37.554 UTC [6997] postgres@postgres LOG:  process 6997 acquired ShareLock on transaction 1067 after 105085.677 ms
2024-03-17 17:16:37.554 UTC [6997] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "my_table"
2024-03-17 17:16:37.554 UTC [6997] postgres@postgres STATEMENT:  UPDATE my_table  SET data = 'new_data' WHERE id = 50001;
```

Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах:


```
postgres=*# UPDATE my_table SET data = 'value from session 1' WHERE id = 50001; --1

postgres=*# UPDATE my_table SET data = 'value from session 2' WHERE id = 50001; --2

postgres=*# UPDATE my_table SET data = 'value from session 3' WHERE id = 50001; --3
```
Посмотрим на pg_locks:

```
postgres=# SELECT locktype, relation::regclass, mode, transactionid, virtualxid, pid, granted
postgres-# FROM pg_locks
postgres-# WHERE relation = 'my_table'::regclass;


```
Вывод:
```
locktype | relation |       mode       | transactionid | virtualxid | pid  | granted 
----------+----------+------------------+---------------+------------+------+---------
 relation | my_table | RowExclusiveLock |               |            | 7381 | t
 relation | my_table | RowExclusiveLock |               |            | 7377 | t
 tuple    | my_table | ExclusiveLock    |               |            | 7374 | f
(5 rows)n | my_table | RowExclusiveLock |               |            | 7381 | t
 relation | my_table | RowExclusiveLock |               |            | 7377 | t
~relation | my_table | RowExclusiveLock |               |            | 7374 | t
~tuple    | my_table | ExclusiveLock    |               |            | 7374 | f
~tuple    | my_table | ExclusiveLock    |               |            | 7377 | t

```
RowExclusiveLock (7381, 7377, 7374, все с granted = t): Три разных процесса удерживают блокировку на уровне таблицы, что указывает на то, что они выполняют или планируют выполнить операции, изменяющие строки. Эти блокировки совместимы друг с другом, так как они позволяют нескольким транзакциям изменять разные строки в таблице одновременно, но не позволяют начать новые транзакции, которые бы конфликтовали с текущими изменениями.
ExclusiveLock на уровне tuple (7374 с granted = f, 7377 с granted = t): Это указывает на то, что процесс с pid 7377 удерживает блокировку на конкретную строку, успешно предотвращая другие транзакции от её изменения, в то время как процесс с pid 7374 пытается получить эксклюзивную блокировку на ту же строку, но пока не может это сделать (granted = f).


Перейдем к взаимоблокировке 3 транзакций:
```
Сеанс 1:
BEGIN; — начало транзакции.
UPDATE deadlock_demo SET value = 'Updated 1' WHERE id = 1; — обновление первой строки.
Сеанс 2:
BEGIN; — начало транзакции.
UPDATE deadlock_demo SET value = 'Updated 2' WHERE id = 2; — обновление второй строки.
Сеанс 3:
BEGIN; — начало транзакции.
UPDATE deadlock_demo SET value = 'Updated 3' WHERE id = 3; — обновление третьей строки.
```

До этого момента взаимная блокировка ещё не произошла. Теперь, чтобы вызвать взаимную блокировку, нужно создать условия, при которых каждая транзакция будет ожидать освобождения ресурса, захваченного другой транзакцией:


```
UPDATE deadlock_demo SET value = 'Updated 1 again' WHERE id = 2; — попытка обновить вторую строку, которая уже заблокирована сеансом 2.
Сеанс 2:
UPDATE deadlock_demo SET value = 'Updated 2 again' WHERE id = 3; — попытка обновить третью строку, которая уже заблокирована сеансом 3.
Сеанс 3:
UPDATE deadlock_demo SET value = 'Updated 3 again' WHERE id = 1; — попытка обновить первую строку, которая уже заблокирована сеансом 1.
```
На этом шаге, если выполнить команды последовательно, одна за другой, каждая из транзакций пытается обновить строку, уже заблокированную другой транзакцией, создавая цикл ожидания без возможности его разрешения.

Посмотрим логи:
```
2024-03-17 18:33:11.311 UTC [7891] postgres@postgres LOG:  process 7891 still waiting for ShareLock on transaction 1080 after 200.160 ms
2024-03-17 18:33:11.311 UTC [7891] postgres@postgres DETAIL:  Process holding the lock: 7910. Wait queue: 7891.
2024-03-17 18:33:11.311 UTC [7891] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "deadlock_demo"
2024-03-17 18:33:11.311 UTC [7891] postgres@postgres STATEMENT:  UPDATE deadlock_demo SET value = 'Updated 1 again' WHERE id = 2;
2024-03-17 18:33:49.272 UTC [7910] postgres@postgres LOG:  process 7910 still waiting for ShareLock on transaction 1081 after 200.105 ms
2024-03-17 18:33:49.272 UTC [7910] postgres@postgres DETAIL:  Process holding the lock: 7915. Wait queue: 7910.
2024-03-17 18:33:49.272 UTC [7910] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "deadlock_demo"
2024-03-17 18:33:49.272 UTC [7910] postgres@postgres STATEMENT:  UPDATE deadlock_demo SET value = 'Updated 2 again' WHERE id = 3;
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres LOG:  process 7915 detected deadlock while waiting for ShareLock on transaction 1079 after 200.275 ms
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres DETAIL:  Process holding the lock: 7891. Wait queue: .
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "deadlock_demo"
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres STATEMENT:  UPDATE deadlock_demo SET value = 'Updated 3 again' WHERE id = 1;
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres ERROR:  deadlock detected
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres DETAIL:  Process 7915 waits for ShareLock on transaction 1079; blocked by process 7891.
	Process 7891 waits for ShareLock on transaction 1080; blocked by process 7910.
	Process 7910 waits for ShareLock on transaction 1081; blocked by process 7915.
	Process 7915: UPDATE deadlock_demo SET value = 'Updated 3 again' WHERE id = 1;
	Process 7891: UPDATE deadlock_demo SET value = 'Updated 1 again' WHERE id = 2;
	Process 7910: UPDATE deadlock_demo SET value = 'Updated 2 again' WHERE id = 3;
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres HINT:  See server log for query details.
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "deadlock_demo"
2024-03-17 18:34:08.400 UTC [7915] postgres@postgres STATEMENT:  UPDATE deadlock_demo SET value = 'Updated 3 again' WHERE id = 1;
2024-03-17 18:34:08.400 UTC [7910] postgres@postgres LOG:  process 7910 acquired ShareLock on transaction 1081 after 19328.452 ms
2024-03-17 18:34:08.400 UTC [7910] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "deadlock_demo"
2024-03-17 18:34:08.400 UTC [7910] postgres@postgres STATEMENT:  UPDATE deadlock_demo SET value = 'Updated 2 again' WHERE id = 3;
2024-03-17 18:34:49.793 UTC [7891] postgres@postgres LOG:  process 7891 acquired ShareLock on transaction 1080 after 98682.214 ms
2024-03-17 18:34:49.793 UTC [7891] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "deadlock_demo"
2024-03-17 18:34:49.793 UTC [7891] postgres@postgres STATEMENT:  UPDATE deadlock_demo SET value = 'Updated 1 again' WHERE id = 2;
```
По логу мы видим, как именно мы поймали дедлок. Восстановить полную картину впервые видя лог довольно тяжело, помог селект:


```
postgres=# select * from deadlock_demo;
 id |      value      
----+-----------------
  1 | Updated 1
  2 | Updated 2
  3 | Updated 2 again
```
Прошла только вторая транзакция, прервались 1 и 3 транзакции.

Ответ на 4 вопрос: На стандартных настройках блокировок скорее всего нет, т.к. UPDATE в первой транзакции залочит всю таблицу (потому что без WHERE)


