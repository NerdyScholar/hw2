# Homework 2


В первой сессии создаем таблицу и наполняем ее данными:

```
  create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```


Результат:

```
  CREATE TABLE
INSERT 0 1
INSERT 0 1
WARNING:  there is no transaction in progress
COMMIT
```



Смотрим текущий уровень изоляции:

```
  show transaction isolation level
```


Результат:

<img width="258" alt="Снимок экрана 2024-02-15 в 21 30 08" src="https://github.com/NerdyScholar/hw2/assets/160164728/190b024f-93df-4007-9335-e1021062ddcb">



Вставим в первой сессии новую строку:

```
  BEGIN
otus_db=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```



Посмотрим на таблицу через вторую сессию:
```
  BEGIN
otus_db=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```



Мы не видим новую запись, так как при уровне изоляции *read_committed* нам нужно сначала закомитить изменения в первой сессии, чтобы увидеть их во второй



Заселектим таблицу после коммита в первой сессии:

```
  otus_db=*# select * from persons;
  id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
(3 rows)
```



Мы видим новую строчку из-за того, что мы закоммитили наш инсерт



Повторяем наши действия, но теперь с уровнем изоляции *repeatable_read*

```
  otus_db=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```



Результат селекта в другой сессии:

```

  otus_db=# begin;
BEGIN
otus_db=*# set transaction isolation level repeatable read;
SET
otus_db=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
  6 | sergey     | sergeeivich
(4 rows)
```


Мы не видим строчку, добавленную в первой сессии, так как Уровень изоляции "Repeatable Read"  использует блокировки чтения для предотвращения чтения данных, которые были изменены другими транзакциями, но ещё не были зафиксированы. Однако незакомиченные изменения из других транзакций  не видны именно потому, что предоставляется консистентное (постоянное) чтение.



Закоммитим в первой сессии и повторим селект во второй:

```
  otus_db=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
  6 | sergey     | sergeeivich
(4 rows)
```


Новые данные все еще не видны, так как при текущем уровне изоляции нам нужно закончить транзакцию, чтобы увидеть изменения из других сессий



Закоммитим во второй сессии и повторим селект в ней:

```
  otus_db=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
  6 | sergey     | sergeeivich
(4 rows)


otus_db=*# commit;
COMMIT
otus_db=# begin;
BEGIN
otus_db=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
  6 | sergey     | sergeeivich
  7 | sveta      | svetova
(5 rows)
```


Мы видим новую строчку, так как завершили предыдущую транзакцию, вторая сессия не могла получить  изменения первой сессии до того, как не была завершена предыдущаю транзакция repeatable_read


