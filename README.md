1. Для проведения оптимизации БД была взята демонстрационная база авиперелетов РФ: https://postgrespro.ru/education/demodb, а именно большая версия на 2.5 гб demo-big.zip 

2. Была написана функция генерации новых записей в новой схеме данных демонстрационной БД, чтобы проводить тестирование не только на БД размером 2.5 ГБ, но и на БД размером около 35 ГБ

```
-- DROP FUNCTION bookings_huge.bookings_autofill();

CREATE OR REPLACE FUNCTION bookings_huge.bookings_autofill()
 RETURNS void
 LANGUAGE plpgsql
AS $function$
DECLARE
    aircraft_codes text[] := array['A32', 'B73', 'C12'];
    airports text[] := array['JFK', 'LAX', 'SFO', 'ORD', 'ATL'];
    fare_conditions text[] := array['Economy', 'Comfort', 'Business'];
    start_date timestamp := '2023-01-01 00:00:00';
    end_date timestamp := '2024-01-01 00:00:00';
    total_records int := 10000000; -- указывается желаемое кол-во записей
    i int;
    j int;
    book_ref text;
    ticket_no text;
    flight_id int;
    book_date timestamp;
    scheduled_departure timestamp;
    scheduled_arrival timestamp;
BEGIN
    -- Заполнение таблицы aircrafts_data
    FOR i IN 1..array_length(aircraft_codes, 1) LOOP
        INSERT INTO bookings_huge.aircrafts_data (aircraft_code, model, range)
        VALUES (aircraft_codes[i], jsonb_build_object('en', 'Model ' || aircraft_codes[i]), 5000 + i * 1000);
    END LOOP;

    -- Заполнение таблицы airports_data
    FOR i IN 1..array_length(airports, 1) LOOP
        INSERT INTO bookings_huge.airports_data (airport_code, airport_name, city, coordinates, timezone)
        VALUES (airports[i], jsonb_build_object('en', 'Airport ' || airports[i]), jsonb_build_object('en', 'City ' || airports[i]), point(40.0 + i, -74.0 + i), 'UTC');
    END LOOP;

    -- Заполнение таблицы bookings и связанной с ней tickets
    FOR i IN 1..total_records LOOP
        book_ref := lpad(i::text, 10, '0');
        book_date := start_date + random() * (end_date - start_date);

        INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
        VALUES (book_ref, book_date, round((random() * 1000)::numeric, 2));

        ticket_no := lpad((i * 10)::text, 13, '0');

        INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name, contact_data)
        VALUES (ticket_no, book_ref, 'ID' || i, 'Passenger ' || i, jsonb_build_object('phone', '1234567890'));

        FOR j IN 1..5 LOOP
            flight_id := nextval('bookings_huge.flights_flight_id_seq');
            scheduled_departure := start_date + random() * (end_date - start_date);
            scheduled_arrival := scheduled_departure + interval '1 hour' * (1 + random() * 5);

            INSERT INTO bookings_huge.flights (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code)
            VALUES (flight_id, lpad(flight_id::text, 6, '0'), scheduled_departure, scheduled_arrival, airports[j % array_length(airports, 1) + 1], airports[(j + 1) % array_length(airports, 1) + 1], 'Scheduled', aircraft_codes[j % array_length(aircraft_codes, 1) + 1]);

            INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
            VALUES (ticket_no, flight_id, fare_conditions[j % array_length(fare_conditions, 1) + 1], round((random() * 500)::numeric, 2));

            INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
            VALUES (ticket_no, flight_id, j, 'A' || j);
        END LOOP;
    END LOOP;
END;
$function$
;


```


3. Был написан кастомный скрипт для создания "полезной" нагрузки на обе схемы данных (bookings - 2.5 гб, bookings_huge = ~35 гб):


```
   -- Создаем рандомные значения для столбцов
\set book_ref random(100000, 999999)
\set ticket_no random(1000000000000, 9999999999999)
\set passenger_id random(1000, 9999)
\set boarding_no random(1, 200)
\set seat_no random(1000, 9999)
\set new_seat_no random(1000, 9999)
\set amount random(100, 10000) / 100.0
\set new_amount random(100, 10000) / 100.0
\set total_amount random(100, 10000) / 100.0
\set flight_id random(1, 200000)

-- Вставка в bookings.bookings
BEGIN;
INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
VALUES (LPAD(:book_ref::text, 11, '0'), CURRENT_TIMESTAMP, :total_amount);
END;

-- Выборка вставленной строки из bookings
BEGIN;
SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
END;

-- Вставка в bookings.tickets
BEGIN;
INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
VALUES (LPAD(:ticket_no::text, 14, '0'), LPAD(:book_ref::text, 11, '0'), 'P' || :passenger_id::text, 'Иван Иванович');
END;

--  Выборка вставленной строки из tickets
BEGIN;
SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
END;

--Вставка в bookings.ticket_flights
BEGIN;
INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
VALUES (LPAD(:ticket_no::text, 14, '0'), :flight_id, 'Economy', :amount);
END;

--Выборка вставленной строки из ticket_flights
BEGIN;
SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

-- Вставка в bookings.boarding_passes
BEGIN;
INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
VALUES (LPAD(:ticket_no::text, 14, '0'), :flight_id, :boarding_no, :seat_no::text);
END;

-- Выборка вставленной строки из boarding_passes
BEGIN;
SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

-- Обновление строки в bookings.boarding_passes
BEGIN;
UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

-- Выборка вставленной строки из boarding_passes
BEGIN;
SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

-- Обновление строки в bookings.ticket_flights
\set old_amount :amount
BEGIN;
UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

-- Выборка вставленной строки из ticket_flights
BEGIN;
SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

-- Обновление строки в bookings.tickets
BEGIN;
UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
END;

-- Выборка вставленной строки из tickets
BEGIN;
SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
END;

-- Обновление строки в bookings.bookings
BEGIN;
UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 11, '0');
END;

-- Выборка вставленной строки из bookings
BEGIN;
SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
END;

-- Удаление сгенерированных строк
BEGIN;
DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

BEGIN;
DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
END;

BEGIN;
DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
END;

BEGIN;
DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
END;


   ```


4. Затем было проведено тестирование "ванильной" PostgreSQL на всех настройках по умолчанию на двух схемах с помощью утилиты pgbench и написанного скрипта, сначала в однопоточном режиме:
   ```
   pgbench -c 1 -T 900 -U postgres -h localhost -d -r -f bookings_bench32.sql demo -n
   ```

Результаты: 
```
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 1800 s
number of transactions actually processed: 4299
number of failed transactions: 0 (0.000%)
latency average = 418.783 ms
initial connection time = 16.997 ms
tps = 2.387870 (without initial connection time)
statement latencies in milliseconds and failures:
         0.016           0  \set book_ref random(100000, 999999)
         0.011           0  \set ticket_no random(1000000000000, 9999999999999)
         0.011           0  \set passenger_id random(1000, 9999)
         0.012           0  \set boarding_no random(1, 200)
         0.012           0  \set seat_no random(1000, 9999)
         0.012           0  \set new_seat_no random(1000, 9999)
         0.013           0  \set amount random(100, 10000) / 100.0
         0.013           0  \set new_amount random(100, 10000) / 100.0
         0.013           0  \set total_amount random(100, 10000) / 100.0
         0.012           0  \set flight_id random(1, 200000)
         0.185           0  BEGIN;
         0.482           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         3.613           0  END;
         0.138           0  BEGIN;
         0.340           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.117           0  END;
         0.122           0  BEGIN;
         0.436           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         3.485           0  END;
         0.130           0  BEGIN;
         0.291           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.120           0  END;
         0.110           0  BEGIN;
         2.730           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         3.435           0  END;
         0.129           0  BEGIN;
         0.289           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.119           0  END;
         0.104           0  BEGIN;
         0.266           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         3.320           0  END;
         0.124           0  BEGIN;
         0.271           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.116           0  END;
         0.103           0  BEGIN;
         0.265           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
         3.225           0  END;
         0.124           0  BEGIN;
         0.260           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.117           0  END;
         0.018           0  \set old_amount :amount
         0.105           0  BEGIN;
         0.303           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
         3.222           0  END;
         0.123           0  BEGIN;
         0.265           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.117           0  END;
         0.105           0  BEGIN;
         0.249           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         3.191           0  END;
         0.123           0  BEGIN;
         0.253           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.118           0  END;
         0.104           0  BEGIN;
         0.279           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         3.250           0  END;
         0.123           0  BEGIN;
         0.256           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.117           0  END;
         0.105           0  BEGIN;
         0.245           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         3.163           0  END;
         0.122           0  BEGIN;
         0.317           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         3.093           0  END;
         0.124           0  BEGIN;
         0.308           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         3.096           0  END;
         0.125           0  BEGIN;
       366.795           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         4.262           0  END;


```
Для 35 гб схемы:
```
 transaction type: bookings_huge_bench.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 1800 s
number of transactions actually processed: 1334
number of failed transactions: 0 (0.000%)
latency average = 1350.127 ms
initial connection time = 15.956 ms
tps = 0.740671 (without initial connection time)
statement latencies in milliseconds and failures:
         0.016           0  \set book_ref random(100000, 999999)
         0.012           0  \set ticket_no random(1000000000000, 9999999999999)
         0.011           0  \set passenger_id random(1000, 9999)
         0.012           0  \set boarding_no random(1, 200)
         0.012           0  \set seat_no random(1000, 9999)
         0.012           0  \set new_seat_no random(1000, 9999)
         0.013           0  \set amount random(100, 10000) / 100.0
         0.013           0  \set new_amount random(100, 10000) / 100.0
         0.012           0  \set total_amount random(100, 10000) / 100.0
         0.012           0  \set flight_id random(1, 200000)
         0.177           0  BEGIN;
         7.298           0  INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
         4.096           0  END;
         0.127           0  BEGIN;
         0.305           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.113           0  END;
         0.098           0  BEGIN;
         0.370           0  INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         3.989           0  END;
         0.121           0  BEGIN;
         0.246           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.111           0  END;
         0.095           0  BEGIN;
        38.087           0  INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         4.208           0  END;
         0.135           0  BEGIN;
         0.316           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.117           0  END;
         0.109           0  BEGIN;
         0.801           0  INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         3.811           0  END;
         0.121           0  BEGIN;
         0.278           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.113           0  END;
         0.104           0  BEGIN;
         0.670           0  UPDATE bookings_huge.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND fli
         3.627           0  END;
         0.117           0  BEGIN;
         0.245           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.112           0  END;
         0.019           0  \set old_amount :amount
         0.098           0  BEGIN;
         0.284           0  UPDATE bookings_huge.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = 
         3.575           0  END;
         0.119           0  BEGIN;
         0.244           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.112           0  END;
         0.097           0  BEGIN;
         0.231           0  UPDATE bookings_huge.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         3.493           0  END;
         0.119           0  BEGIN;
         0.234           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.112           0  END;
         0.099           0  BEGIN;
         0.260           0  UPDATE bookings_huge.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::te
         3.453           0  END;
         0.118           0  BEGIN;
         0.240           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.114           0  END;
         0.099           0  BEGIN;
         0.232           0  DELETE FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         3.466           0  END;
         0.116           0  BEGIN;
         0.301           0  DELETE FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         3.278           0  END;
         0.118           0  BEGIN;
        28.365           0  DELETE FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.334           0  END;
         0.142           0  BEGIN;
      1221.804           0  DELETE FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         4.598           0  END;


```

5. тесты c 10 клиентами:
```
pgbench -c 10 -T 900 -U postgres -h localhost -d -r -f bookings_bench32.sql demo -n


```

Результаты: 
```
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 5430
number of failed transactions: 0 (0.000%)
latency average = 1659.289 ms
initial connection time = 154.875 ms
tps = 6.026680 (without initial connection time)
statement latencies in milliseconds and failures:
         0.014           0  \set book_ref random(100000, 999999)
         0.009           0  \set ticket_no random(1000000000000, 9999999999999)
         0.009           0  \set passenger_id random(1000, 9999)
         0.009           0  \set boarding_no random(1, 200)
         0.009           0  \set seat_no random(1000, 9999)
         0.008           0  \set new_seat_no random(1000, 9999)
         0.009           0  \set amount random(100, 10000) / 100.0
         0.009           0  \set new_amount random(100, 10000) / 100.0
         0.009           0  \set total_amount random(100, 10000) / 100.0
         0.009           0  \set flight_id random(1, 200000)
         0.610           0  BEGIN;
         0.510           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         5.808           0  END;
         0.243           0  BEGIN;
         0.385           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.226           0  END;
         0.174           0  BEGIN;
         0.435           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         5.393           0  END;
         0.218           0  BEGIN;
         0.344           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.206           0  END;
         0.171           0  BEGIN;
         0.496           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         5.221           0  END;
         0.202           0  BEGIN;
         0.362           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.203           0  END;
         0.172           0  BEGIN;
         0.358           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         5.001           0  END;
         0.204           0  BEGIN;
         0.348           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.195           0  END;
         0.164           0  BEGIN;
         0.343           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
         4.767           0  END;
         0.190           0  BEGIN;
         0.324           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.195           0  END;
         0.013           0  \set old_amount :amount
         0.165           0  BEGIN;
         0.379           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
         4.878           0  END;
         0.205           0  BEGIN;
         0.330           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.198           0  END;
         0.161           0  BEGIN;
         0.342           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.831           0  END;
         0.204           0  BEGIN;
         0.318           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.200           0  END;
         0.172           0  BEGIN;
         0.341           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         4.717           0  END;
         0.188           0  BEGIN;
         0.317           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.194           0  END;
         0.163           0  BEGIN;
         0.303           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         4.569           0  END;
         0.200           0  BEGIN;
         0.400           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         4.696           0  END;
         0.194           0  BEGIN;
         0.383           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.669           0  END;
         0.200           0  BEGIN;
      1577.706           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
        13.479           0  END;



```


```
transaction type: bookings_huge_bench.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 1711
number of failed transactions: 0 (0.000%)
latency average = 5277.744 ms
initial connection time = 151.407 ms
tps = 1.894749 (without initial connection time)
statement latencies in milliseconds and failures:
         0.014           0  \set book_ref random(100000, 999999)
         0.009           0  \set ticket_no random(1000000000000, 9999999999999)
         0.008           0  \set passenger_id random(1000, 9999)
         0.009           0  \set boarding_no random(1, 200)
         0.008           0  \set seat_no random(1000, 9999)
         0.009           0  \set new_seat_no random(1000, 9999)
         0.009           0  \set amount random(100, 10000) / 100.0
         0.009           0  \set new_amount random(100, 10000) / 100.0
         0.008           0  \set total_amount random(100, 10000) / 100.0
         0.008           0  \set flight_id random(1, 200000)
         0.629           0  BEGIN;
         0.548           0  INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
         5.703           0  END;
         0.260           0  BEGIN;
         0.452           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.235           0  END;
         0.192           0  BEGIN;
         0.405           0  INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         5.045           0  END;
         0.211           0  BEGIN;
         0.350           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.193           0  END;
         0.155           0  BEGIN;
        10.820           0  INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         4.978           0  END;
         0.209           0  BEGIN;
         0.380           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.211           0  END;
         0.169           0  BEGIN;
         0.401           0  INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         4.613           0  END;
         0.194           0  BEGIN;
         0.351           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.184           0  END;
         0.153           0  BEGIN;
         0.341           0  UPDATE bookings_huge.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND fli
         4.261           0  END;
         0.213           0  BEGIN;
         0.316           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.193           0  END;
         0.013           0  \set old_amount :amount
         0.155           0  BEGIN;
         0.381           0  UPDATE bookings_huge.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = 
         4.360           0  END;
         0.190           0  BEGIN;
         0.325           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.190           0  END;
         0.160           0  BEGIN;
         0.315           0  UPDATE bookings_huge.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.540           0  END;
         0.203           0  BEGIN;
         0.309           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.199           0  END;
         0.155           0  BEGIN;
         0.329           0  UPDATE bookings_huge.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::te
         4.604           0  END;
         0.207           0  BEGIN;
         0.320           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.195           0  END;
         0.168           0  BEGIN;
         0.290           0  DELETE FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         4.527           0  END;
         0.213           0  BEGIN;
         0.419           0  DELETE FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         4.428           0  END;
         0.202           0  BEGIN;
        79.651           0  DELETE FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
        12.639           0  END;
         0.740           0  BEGIN;
      5096.062           0  DELETE FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
        13.473           0  END;




```
6. Тест производительности с 75 клиентами:

```
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 75
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 5499
number of failed transactions: 0 (0.000%)
latency average = 12344.533 ms
initial connection time = 1147.155 ms
tps = 6.075564 (without initial connection time)
statement latencies in milliseconds and failures:
         0.015           0  \set book_ref random(100000, 999999)
         0.010           0  \set ticket_no random(1000000000000, 9999999999999)
         0.009           0  \set passenger_id random(1000, 9999)
         0.009           0  \set boarding_no random(1, 200)
         0.009           0  \set seat_no random(1000, 9999)
         0.009           0  \set new_seat_no random(1000, 9999)
         0.010           0  \set amount random(100, 10000) / 100.0
         0.010           0  \set new_amount random(100, 10000) / 100.0
         0.010           0  \set total_amount random(100, 10000) / 100.0
         0.009           0  \set flight_id random(1, 200000)
         1.736           0  BEGIN;
         3.782           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
        12.252           0  END;
         1.299           0  BEGIN;
         2.528           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         1.554           0  END;
         1.123           0  BEGIN;
         2.642           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
        12.456           0  END;
         1.268           0  BEGIN;
         2.325           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         1.422           0  END;
         1.078           0  BEGIN;
         4.589           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
        12.729           0  END;
         1.377           0  BEGIN;
         1.833           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         1.505           0  END;
         1.057           0  BEGIN;
         2.598           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
        11.793           0  END;
         1.317           0  BEGIN;
         2.010           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         1.356           0  END;
         1.119           0  BEGIN;
         1.536           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
        11.283           0  END;
         1.155           0  BEGIN;
         1.972           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         1.273           0  END;
         0.016           0  \set old_amount :amount
         1.020           0  BEGIN;
         1.813           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
        11.630           0  END;
         1.176           0  BEGIN;
         1.779           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         1.382           0  END;
         1.041           0  BEGIN;
         2.570           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
        11.268           0  END;
         1.207           0  BEGIN;
         2.083           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         1.265           0  END;
         1.003           0  BEGIN;
         2.074           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
        11.165           0  END;
         1.185           0  BEGIN;
         2.390           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         1.280           0  END;
         0.969           0  BEGIN;
         1.945           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
        11.125           0  END;
         1.110           0  BEGIN;
         2.640           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
        10.921           0  END;
         1.111           0  BEGIN;
         3.059           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
        10.766           0  END;
         1.061           0  BEGIN;
     12083.014           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
        23.161           0  END;



```
Как мы видим, результаты получились ожинаковые (и на большой БД тоже), поэтому было принято решение дальнейшие тесты провожить с 10 клиентами.


7. Далее начались настройки конфига в postgresql.conf
Для начала под "OLTP" нагрузку были изменены настройки памяти, оптимальные значения через несколько прогонов pgbench получились следующие:


```
shared_buffers = '2048 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '6 GB'
effective_io_concurrency = 1 
random_page_cost = 4


```
 Результат тестов: 
 
 ```
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 6238
number of failed transactions: 0 (0.000%)
latency average = 1443.947 ms
initial connection time = 167.977 ms
tps = 6.925461 (without initial connection time)
statement latencies in milliseconds and failures:
         0.014           0  \set book_ref random(100000, 999999)
         0.008           0  \set ticket_no random(1000000000000, 9999999999999)
         0.008           0  \set passenger_id random(1000, 9999)
         0.008           0  \set boarding_no random(1, 200)
         0.008           0  \set seat_no random(1000, 9999)
         0.008           0  \set new_seat_no random(1000, 9999)
         0.009           0  \set amount random(100, 10000) / 100.0
         0.009           0  \set new_amount random(100, 10000) / 100.0
         0.008           0  \set total_amount random(100, 10000) / 100.0
         0.008           0  \set flight_id random(1, 200000)
         0.374           0  BEGIN;
         0.406           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         5.917           0  END;
         0.206           0  BEGIN;
         0.368           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.198           0  END;
         0.169           0  BEGIN;
         0.384           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         5.415           0  END;
         0.192           0  BEGIN;
         0.321           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.186           0  END;
         0.164           0  BEGIN;
         0.448           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         5.163           0  END;
         0.197           0  BEGIN;
         0.343           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.192           0  END;
         0.157           0  BEGIN;
         0.329           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         5.085           0  END;
         0.188           0  BEGIN;
         0.333           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.182           0  END;
         0.155           0  BEGIN;
         0.321           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
         4.913           0  END;
         0.182           0  BEGIN;
         0.318           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.190           0  END;
         0.015           0  \set old_amount :amount
         0.157           0  BEGIN;
         0.372           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
         4.914           0  END;
         0.186           0  BEGIN;
         0.325           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.194           0  END;
         0.159           0  BEGIN;
         0.296           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.867           0  END;
         0.188           0  BEGIN;
         0.310           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.190           0  END;
         0.157           0  BEGIN;
         0.318           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         4.896           0  END;
         0.183           0  BEGIN;
         0.306           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.190           0  END;
         0.160           0  BEGIN;
         0.285           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         4.896           0  END;
         0.186           0  BEGIN;
         0.394           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         4.833           0  END;
         0.185           0  BEGIN;
         0.368           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.968           0  END;
         0.183           0  BEGIN;
      1362.096           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
        13.752           0  END;



```


```
transaction type: bookings_huge_bench.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 1710
number of failed transactions: 0 (0.000%)
latency average = 5279.502 ms
initial connection time = 154.630 ms
tps = 1.894118 (without initial connection time)
statement latencies in milliseconds and failures:
         0.014           0  \set book_ref random(100000, 999999)
         0.009           0  \set ticket_no random(1000000000000, 9999999999999)
         0.008           0  \set passenger_id random(1000, 9999)
         0.008           0  \set boarding_no random(1, 200)
         0.008           0  \set seat_no random(1000, 9999)
         0.008           0  \set new_seat_no random(1000, 9999)
         0.009           0  \set amount random(100, 10000) / 100.0
         0.009           0  \set new_amount random(100, 10000) / 100.0
         0.009           0  \set total_amount random(100, 10000) / 100.0
         0.008           0  \set flight_id random(1, 200000)
         0.654           0  BEGIN;
         0.537           0  INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
         4.699           0  END;
         0.243           0  BEGIN;
         0.431           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.211           0  END;
         0.180           0  BEGIN;
         0.412           0  INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         4.052           0  END;
         0.195           0  BEGIN;
         0.334           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.192           0  END;
         0.158           0  BEGIN;
         0.705           0  INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         4.284           0  END;
         0.190           0  BEGIN;
         0.346           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.196           0  END;
         0.149           0  BEGIN;
         0.385           0  INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         3.851           0  END;
         0.196           0  BEGIN;
         0.317           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.189           0  END;
         0.154           0  BEGIN;
         0.325           0  UPDATE bookings_huge.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND fli
         3.731           0  END;
         0.185           0  BEGIN;
         0.319           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.207           0  END;
         0.014           0  \set old_amount :amount
         0.154           0  BEGIN;
         0.401           0  UPDATE bookings_huge.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = 
         3.753           0  END;
         0.186           0  BEGIN;
         0.319           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.185           0  END;
         0.153           0  BEGIN;
         0.317           0  UPDATE bookings_huge.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         3.781           0  END;
         0.195           0  BEGIN;
         0.297           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.201           0  END;
         0.160           0  BEGIN;
         0.347           0  UPDATE bookings_huge.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::te
         3.678           0  END;
         0.195           0  BEGIN;
         0.304           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.175           0  END;
         0.158           0  BEGIN;
         0.299           0  DELETE FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         3.834           0  END;
         0.193           0  BEGIN;
         0.442           0  DELETE FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         3.854           0  END;
         0.199           0  BEGIN;
        81.692           0  DELETE FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
        12.364           0  END;
         0.787           0  BEGIN;
      5114.110           0  DELETE FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
        12.978           0  END;


--5х после изменения Background writer, Parallel queries, Advanced features
tps = 2.001587 (without initial connection time)
statement latencies in milliseconds and failures:
         0.015           0  \set book_ref random(100000, 999999)
         0.009           0  \set ticket_no random(1000000000000, 9999999999999)
         0.009           0  \set passenger_id random(1000, 9999)
         0.009           0  \set boarding_no random(1, 200)
         0.009           0  \set seat_no random(1000, 9999)
         0.009           0  \set new_seat_no random(1000, 9999)
         0.009           0  \set amount random(100, 10000) / 100.0
         0.009           0  \set new_amount random(100, 10000) / 100.0
         0.009           0  \set total_amount random(100, 10000) / 100.0
         0.008           0  \set flight_id random(1, 200000)
         1.433           0  BEGIN;
         0.928           0  INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
         0.700           0  END;
         0.358           0  BEGIN;
         0.552           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.501           0  END;
         0.245           0  BEGIN;
         0.563           0  INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         0.492           0  END;
         0.275           0  BEGIN;
         0.368           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.353           0  END;
         0.227           0  BEGIN;
         0.847           0  INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         0.800           0  END;
         0.263           0  BEGIN;
         0.406           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.355           0  END;
         0.218           0  BEGIN;
         0.409           0  INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         0.404           0  END;
         0.225           0  BEGIN;
         0.327           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.298           0  END;
         0.229           0  BEGIN;
         0.383           0  UPDATE bookings_huge.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND fli
         0.355           0  END;
         0.207           0  BEGIN;
         0.325           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.285           0  END;
         0.016           0  \set old_amount :amount
         0.220           0  BEGIN;
         0.462           0  UPDATE bookings_huge.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = 
         0.354           0  END;
         0.198           0  BEGIN;
         0.355           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.341           0  END;
         0.226           0  BEGIN;
         0.351           0  UPDATE bookings_huge.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.379           0  END;
         0.197           0  BEGIN;
         0.318           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.282           0  END;
         0.232           0  BEGIN;
         0.392           0  UPDATE bookings_huge.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::te
         0.335           0  END;
         0.229           0  BEGIN;
         0.298           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.323           0  END;
         0.213           0  BEGIN;
         0.391           0  DELETE FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.344           0  END;
         0.229           0  BEGIN;
         0.475           0  DELETE FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.364           0  END;
         0.210           0  BEGIN;
         4.372           0  DELETE FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         4.278           0  END;
         1.404           0  BEGIN;
      4953.921           0  DELETE FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         6.703           0  END;



```

Стало быстрее примерно на 12-14% на маленькой базе и  большой разницы не обнаружилось для большой БД


8. Далее были изменены следующие параметры:

```
# Monitoring
shared_preload_libraries = 'pg_stat_statements' 
track_io_timing=on 
track_functions=pl 

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = off

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'


# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)


```
Эти параметры никак не сказались на производительности:

```
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 6211
number of failed transactions: 0 (0.000%)
latency average = 1450.273 ms
initial connection time = 173.928 ms
tps = 6.895254 (without initial connection time)
statement latencies in milliseconds and failures:
         0.013           0  \set book_ref random(100000, 999999)
         0.008           0  \set ticket_no random(1000000000000, 9999999999999)
         0.008           0  \set passenger_id random(1000, 9999)
         0.008           0  \set boarding_no random(1, 200)
         0.008           0  \set seat_no random(1000, 9999)
         0.008           0  \set new_seat_no random(1000, 9999)
         0.008           0  \set amount random(100, 10000) / 100.0
         0.008           0  \set new_amount random(100, 10000) / 100.0
         0.008           0  \set total_amount random(100, 10000) / 100.0
         0.008           0  \set flight_id random(1, 200000)
         1.591           0  BEGIN;
         0.672           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         0.412           0  END;
         0.224           0  BEGIN;
         0.380           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.353           0  END;
         0.201           0  BEGIN;
         0.451           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         0.432           0  END;
         0.207           0  BEGIN;
         0.305           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.275           0  END;
         0.196           0  BEGIN;
         0.651           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         0.568           0  END;
         0.242           0  BEGIN;
         0.333           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.293           0  END;
         0.199           0  BEGIN;
         0.430           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         0.417           0  END;
         0.222           0  BEGIN;
         0.315           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.305           0  END;
         0.209           0  BEGIN;
         0.335           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
         0.350           0  END;
         0.205           0  BEGIN;
         0.307           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.287           0  END;
         0.015           0  \set old_amount :amount
         0.200           0  BEGIN;
         0.364           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
         0.360           0  END;
         0.235           0  BEGIN;
         0.312           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.300           0  END;
         0.207           0  BEGIN;
         0.319           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.312           0  END;
         0.198           0  BEGIN;
         0.294           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.293           0  END;
         0.200           0  BEGIN;
         0.356           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         0.333           0  END;
         0.213           0  BEGIN;
         0.311           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.277           0  END;
         0.191           0  BEGIN;
         0.323           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.298           0  END;
         0.207           0  BEGIN;
         0.401           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.411           0  END;
         0.237           0  BEGIN;
         0.353           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.363           0  END;
         0.219           0  BEGIN;
      1422.118           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         8.142           0  END;



```

9. Далее были изменены следующие параметры:


```
# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries:
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 1
wal_recycle = on


```

Эти параметры также сказались на производительности в пределах погрешности


10. Очень смущало время выполнения DELETE в таблице bookings, поэтому началась кропотливая работа с навешиванием и ндексов на различные столбцы и тесты производительности, начал с первичных ключей book_ref(bookings) и ticket_no(tickets), получил следующие результаты:

```
CREATE UNIQUE INDEX tickets_pkey ON bookings.tickets USING btree (ticket_no);
CREATE UNIQUE INDEX bookings_pkey ON bookings.bookings USING btree (book_ref);


```

Результаты получились лучше в районе погрешности и проблема с DELETE не решилась: 


```

transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 6508
number of failed transactions: 0 (0.000%)
latency average = 1383.732 ms
initial connection time = 151.134 ms
tps = 7.226835 (without initial connection time)
statement latencies in milliseconds and failures:
         0.013           0  \set book_ref random(100000, 999999)
         0.008           0  \set ticket_no random(1000000000000, 9999999999999)
         0.008           0  \set passenger_id random(1000, 9999)
         0.007           0  \set boarding_no random(1, 200)
         0.008           0  \set seat_no random(1000, 9999)
         0.007           0  \set new_seat_no random(1000, 9999)
         0.008           0  \set amount random(100, 10000) / 100.0
         0.008           0  \set new_amount random(100, 10000) / 100.0
         0.008           0  \set total_amount random(100, 10000) / 100.0
         0.007           0  \set flight_id random(1, 200000)
         1.517           0  BEGIN;
         0.549           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         0.381           0  END;
         0.211           0  BEGIN;
         0.372           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.323           0  END;
         0.201           0  BEGIN;
         0.487           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         0.459           0  END;
         0.216           0  BEGIN;
         0.308           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.293           0  END;
         0.183           0  BEGIN;
        14.981           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         0.482           0  END;
         0.200           0  BEGIN;
         0.346           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.257           0  END;
         0.182           0  BEGIN;
        15.423           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         0.346           0  END;
         0.198           0  BEGIN;
         0.348           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.244           0  END;
         0.173           0  BEGIN;
         0.326           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
         0.259           0  END;
         0.172           0  BEGIN;
         0.299           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.238           0  END;
         0.013           0  \set old_amount :amount
         0.169           0  BEGIN;
         0.356           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
         0.265           0  END;
         0.189           0  BEGIN;
         0.301           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.234           0  END;
         0.184           0  BEGIN;
         0.391           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.290           0  END;
         0.197           0  BEGIN;
         0.275           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.220           0  END;
         0.169           0  BEGIN;
         0.309           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         0.233           0  END;
         0.170           0  BEGIN;
         0.266           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.220           0  END;
         0.175           0  BEGIN;
         0.289           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.257           0  END;
         0.183           0  BEGIN;
         0.360           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.272           0  END;
         0.188           0  BEGIN;
         0.347           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.294           0  END;
         0.202           0  BEGIN;
      1328.441           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         7.780           0  END;



```
11. Далее были добавлены индексы на внешние ключи:

```
-- Индексы для внешнего ключа boarding_passes_ticket_no_fkey
CREATE INDEX idx_boarding_passes_ticket_no ON bookings.boarding_passes (ticket_no);
CREATE INDEX idx_boarding_passes_flight_id ON bookings.boarding_passes (flight_id);

-- Индексы для внешнего ключа flights_aircraft_code_fkey
CREATE INDEX idx_flights_aircraft_code ON bookings_huge.flights (aircraft_code);

-- Индексы для внешнего ключа flights_arrival_airport_fkey
CREATE INDEX idx_flights_arrival_airport ON bookings_huge.flights (arrival_airport);

-- Индексы для внешнего ключа flights_departure_airport_fkey
CREATE INDEX idx_flights_departure_airport ON bookings_huge.flights (departure_airport);

-- Индексы для внешнего ключа seats_aircraft_code_fkey
CREATE INDEX idx_seats_aircraft_code ON bookings_huge.seats (aircraft_code);

-- Индексы для внешнего ключа ticket_flights_flight_id_fkey
CREATE INDEX idx_ticket_flights_flight_id ON bookings.ticket_flights (flight_id);

-- Индексы для внешнего ключа ticket_flights_ticket_no_fkey
CREATE INDEX idx_ticket_flights_ticket_no ON bookings.ticket_flights (ticket_no);

-- Индексы для внешнего ключа tickets_book_ref_fkey
CREATE INDEX idx_tickets_book_ref ON bookings.tickets (book_ref);


```
Это дало самый значительный прирост производительности в разы:

```
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 178922
number of failed transactions: 0 (0.000%)
latency average = 50.295 ms
initial connection time = 159.065 ms
tps = 198.827943 (without initial connection time)
statement latencies in milliseconds and failures:
         0.016           0  \set book_ref random(100000, 999999)
         0.014           0  \set ticket_no random(1000000000000, 9999999999999)
         0.014           0  \set passenger_id random(1000, 9999)
         0.014           0  \set boarding_no random(1, 200)
         0.014           0  \set seat_no random(1000, 9999)
         0.014           0  \set new_seat_no random(1000, 9999)
         0.014           0  \set amount random(100, 10000) / 100.0
         0.014           0  \set new_amount random(100, 10000) / 100.0
         0.014           0  \set total_amount random(100, 10000) / 100.0
         0.013           0  \set flight_id random(1, 200000)
         0.768           0  BEGIN;
         0.857           0  INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
         0.759           0  END;
         0.618           0  BEGIN;
         0.907           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.736           0  END;
         0.604           0  BEGIN;
         1.065           0  INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         0.898           0  END;
         0.645           0  BEGIN;
         0.956           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.724           0  END;
         0.620           0  BEGIN;
         1.322           0  INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         0.944           0  END;
         0.695           0  BEGIN;
         1.078           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.781           0  END;
         0.644           0  BEGIN;
         1.083           0  INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         0.883           0  END;
         0.669           0  BEGIN;
         0.971           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.806           0  END;
         0.672           0  BEGIN;
         0.992           0  UPDATE bookings_huge.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND fli
         0.864           0  END;
         0.646           0  BEGIN;
         0.947           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.745           0  END;
         0.023           0  \set old_amount :amount
         0.633           0  BEGIN;
         1.093           0  UPDATE bookings_huge.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = 
         0.908           0  END;
         0.654           0  BEGIN;
         1.017           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.793           0  END;
         0.627           0  BEGIN;
         0.971           0  UPDATE bookings_huge.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.810           0  END;
         0.656           0  BEGIN;
         0.890           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.815           0  END;
         0.609           0  BEGIN;
         1.019           0  UPDATE bookings_huge.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::te
         0.822           0  END;
         0.620           0  BEGIN;
         0.911           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.717           0  END;
         0.636           0  BEGIN;
         0.953           0  DELETE FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.823           0  END;
         0.655           0  BEGIN;
         1.104           0  DELETE FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.886           0  END;
         0.658           0  BEGIN;
         1.122           0  DELETE FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.877           0  END;
         0.682           0  BEGIN;
         1.113           0  DELETE FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.888           0  END;



```
Для большой БД:

```
transaction type: bookings_huge_bench.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 165484
number of failed transactions: 0 (0.000%)
latency average = 54.379 ms
initial connection time = 165.102 ms
tps = 183.895616 (without initial connection time)
statement latencies in milliseconds and failures:
         0.016           0  \set book_ref random(100000, 999999)
         0.014           0  \set ticket_no random(1000000000000, 9999999999999)
         0.014           0  \set passenger_id random(1000, 9999)
         0.014           0  \set boarding_no random(1, 200)
         0.014           0  \set seat_no random(1000, 9999)
         0.013           0  \set new_seat_no random(1000, 9999)
         0.014           0  \set amount random(100, 10000) / 100.0
         0.014           0  \set new_amount random(100, 10000) / 100.0
         0.013           0  \set total_amount random(100, 10000) / 100.0
         0.013           0  \set flight_id random(1, 200000)
         0.742           0  BEGIN;
         1.123           0  INSERT INTO bookings_huge.bookings (book_ref, book_date, total_amount)
         0.800           0  END;
         0.617           0  BEGIN;
         0.887           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.732           0  END;
         0.673           0  BEGIN;
         1.169           0  INSERT INTO bookings_huge.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         0.901           0  END;
         0.633           0  BEGIN;
         0.883           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.707           0  END;
         0.637           0  BEGIN;
         2.506           0  INSERT INTO bookings_huge.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         0.934           0  END;
         0.664           0  BEGIN;
         1.400           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.843           0  END;
         0.623           0  BEGIN;
         1.259           0  INSERT INTO bookings_huge.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         0.860           0  END;
         0.692           0  BEGIN;
         1.672           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.811           0  END;
         0.672           0  BEGIN;
         1.185           0  UPDATE bookings_huge.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND fli
         0.894           0  END;
         0.732           0  BEGIN;
         1.086           0  SELECT * FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.861           0  END;
         0.019           0  \set old_amount :amount
         0.707           0  BEGIN;
         1.150           0  UPDATE bookings_huge.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = 
         0.864           0  END;
         0.662           0  BEGIN;
         1.014           0  SELECT * FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.738           0  END;
         0.609           0  BEGIN;
         0.952           0  UPDATE bookings_huge.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.797           0  END;
         0.638           0  BEGIN;
         1.053           0  SELECT * FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.781           0  END;
         0.744           0  BEGIN;
         1.080           0  UPDATE bookings_huge.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::te
         0.843           0  END;
         0.691           0  BEGIN;
         0.906           0  SELECT * FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.730           0  END;
         0.598           0  BEGIN;
         1.061           0  DELETE FROM bookings_huge.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.900           0  END;
         0.669           0  BEGIN;
         1.192           0  DELETE FROM bookings_huge.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.993           0  END;
         0.676           0  BEGIN;
         1.076           0  DELETE FROM bookings_huge.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.865           0  END;
         0.637           0  BEGIN;
         1.174           0  DELETE FROM bookings_huge.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         1.075           0  END;

```
Как мы видим, результат стал еще лучше по отношению к предыдущим результатам около 2 транзакций в секунду. Получили прирост в 27 раз для маленькой БД и в ~80 раз для большой БД 


12. Далее был определен именно тот индекс, который и дал заветный прирост:

    ```
    CREATE INDEX idx_tickets_book_ref ON bookings.tickets (book_ref);

    
    ```

    13. Затем было принято решение попробовать секционировать самые объемные таблицы tickets, ticket_flights и boarding_passes:
   
```
        -- Создание главной таблицы tickets
CREATE TABLE bookings.tickets_p (
    ticket_no varchar(14) NOT NULL,
    book_ref varchar(11) NOT NULL,
    passenger_id varchar(20) NOT NULL,
    passenger_name text NOT NULL,
    contact_data jsonb NULL,
    CONSTRAINT tickets_p_pkey PRIMARY KEY (ticket_no, book_ref),
    CONSTRAINT tickets_p_book_ref_fkey FOREIGN KEY (book_ref) REFERENCES bookings.bookings(book_ref)
) PARTITION BY RANGE (ticket_no);

```

```
-- Создание партиций по диапазону значений ticket_no
CREATE TABLE bookings.tickets_p1 PARTITION OF bookings.tickets_p 
    FOR VALUES FROM ('00050000000000') TO ('01250000000000');

CREATE TABLE bookings.tickets_p2 PARTITION OF bookings.tickets_p 
    FOR VALUES FROM ('01250000000000') TO ('02500000000000');

CREATE TABLE bookings.tickets_p3 PARTITION OF bookings.tickets_p 
    FOR VALUES FROM ('02500000000000') TO ('03750000000000');

CREATE TABLE bookings.tickets_p4 PARTITION OF bookings.tickets_p 
    FOR VALUES FROM ('03750000000000') TO ('05000000000000');
   CREATE TABLE bookings.tickets_p5 PARTITION OF bookings.tickets_p
    DEFAULT;

 ```

 ```
-- Создание индексов на партициях
CREATE INDEX idx_tickets_p1_book_ref ON bookings.tickets_p1 (book_ref);
CREATE INDEX idx_tickets_p2_book_ref ON bookings.tickets_p2 (book_ref);
CREATE INDEX idx_tickets_p3_book_ref ON bookings.tickets_p3 (book_ref);
CREATE INDEX idx_tickets_p4_book_ref ON bookings.tickets_p4 (book_ref);
CREATE INDEX idx_tickets_p5_book_ref ON bookings.tickets_p5 (book_ref);
CREATE INDEX idx_tickets_p_book_ref ON bookings.tickets_p (book_ref);

 ```
 
 ```
 -- Перенос данных из старой таблицы в новую секционированную таблицу
INSERT INTO bookings.tickets_p (ticket_no, book_ref, passenger_id, passenger_name, contact_data)
SELECT ticket_no, book_ref, passenger_id, passenger_name, contact_data
FROM bookings.tickets;

 ```
 ```
-- Создание главной таблицы ticket_flights_p
CREATE TABLE bookings.ticket_flights_p (
    ticket_no varchar(14) NOT NULL,
    flight_id int4 NOT NULL,
    fare_conditions varchar(10) NOT NULL,
    amount numeric(10, 2) NOT NULL,
    CONSTRAINT ticket_flights_p_pkey PRIMARY KEY (ticket_no, flight_id),
    CONSTRAINT ticket_flights_p_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES bookings.flights(flight_id),
    CONSTRAINT ticket_flights_p_ticket_no_fkey FOREIGN KEY (ticket_no) REFERENCES bookings.tickets_p(ticket_no)
) PARTITION BY RANGE (flight_id);


 ```
 ```
-- Создание партиций по диапазону значений flight_id
CREATE TABLE bookings.ticket_flights_p1 PARTITION OF bookings.ticket_flights_p
    FOR VALUES FROM (1) TO (83334);

CREATE TABLE bookings.ticket_flights_p2 PARTITION OF bookings.ticket_flights_p
    FOR VALUES FROM (83334) TO (166667);

CREATE TABLE bookings.ticket_flights_p3 PARTITION OF bookings.ticket_flights_p
    FOR VALUES FROM (166667) TO (250000);

-- Партиция по умолчанию для всех других значений flight_id
CREATE TABLE bookings.ticket_flights_p4 PARTITION OF bookings.ticket_flights_p
    DEFAULT;

 ```
```
-- Создание индексов для главной таблицы
CREATE INDEX idx_ticket_flights_p_ticket_no ON bookings.ticket_flights_p (ticket_no);
CREATE INDEX idx_ticket_flights_p_flight_id ON bookings.ticket_flights_p (flight_id);

-- Создание индексов для партиций
CREATE INDEX idx_ticket_flights_p1_ticket_no ON bookings.ticket_flights_p1 (ticket_no);
CREATE INDEX idx_ticket_flights_p1_flight_id ON bookings.ticket_flights_p1 (flight_id);

CREATE INDEX idx_ticket_flights_p2_ticket_no ON bookings.ticket_flights_p2 (ticket_no);
CREATE INDEX idx_ticket_flights_p2_flight_id ON bookings.ticket_flights_p2 (flight_id);

CREATE INDEX idx_ticket_flights_p3_ticket_no ON bookings.ticket_flights_p3 (ticket_no);
CREATE INDEX idx_ticket_flights_p3_flight_id ON bookings.ticket_flights_p3 (flight_id);

CREATE INDEX idx_ticket_flights_p4_ticket_no ON bookings.ticket_flights_p4 (ticket_no);
CREATE INDEX idx_ticket_flights_p4_flight_id ON bookings.ticket_flights_p4 (flight_id);


```
```
-- Перенос данных из старой таблицы в новую секционированную таблицу
INSERT INTO bookings.ticket_flights_p (ticket_no, flight_id, fare_conditions, amount)
SELECT ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights;


```
```
-- Создание главной таблицы boarding_passes
CREATE TABLE bookings.boarding_passes_p (
    ticket_no varchar(14) NOT NULL,
    flight_id int4 NOT NULL,
    boarding_no int4 NOT NULL,
    seat_no varchar(4) NOT NULL,
    CONSTRAINT boarding_passes_p_pkey PRIMARY KEY (ticket_no, flight_id),
    CONSTRAINT boarding_passes_ticket_no_fkey FOREIGN KEY (ticket_no, flight_id) REFERENCES bookings.ticket_flights(ticket_no, flight_id)
) PARTITION BY RANGE (flight_id);


```

```
-- Создание партиций по диапазону значений flight_id
CREATE TABLE bookings.boarding_passes_p1 PARTITION OF bookings.boarding_passes_p
    FOR VALUES FROM (1) TO (83334);

CREATE TABLE bookings.boarding_passes_p2 PARTITION OF bookings.boarding_passes_p
    FOR VALUES FROM (83334) TO (166667);

CREATE TABLE bookings.boarding_passes_p3 PARTITION OF bookings.boarding_passes_p
    FOR VALUES FROM (166667) TO (250000);

-- Партиция по умолчанию для всех других значений flight_id
CREATE TABLE bookings.boarding_passes_p4 PARTITION OF bookings.boarding_passes_p
    DEFAULT;


```

```
-- Создание индексов для главной таблицы
CREATE INDEX idx_boarding_passes_p_ticket_no ON bookings.boarding_passes_p (ticket_no);
CREATE INDEX idx_boarding_passes_p_flight_id ON bookings.boarding_passes_p USING btree (flight_id);

-- Создание индексов для партиций
CREATE INDEX idx_boarding_passes_p1_ticket_no ON bookings.boarding_passes_p1 (ticket_no);
CREATE INDEX idx_boarding_passes_p1_flight_id ON bookings.boarding_passes_p1 USING btree (flight_id);

CREATE INDEX idx_boarding_passes_p2_ticket_no ON bookings.boarding_passes_p2 (ticket_no);
CREATE INDEX idx_boarding_passes_p2_flight_id ON bookings.boarding_passes_p2 USING btree (flight_id);

CREATE INDEX idx_boarding_passes_p3_ticket_no ON bookings.boarding_passes_p3 (ticket_no);
CREATE INDEX idx_boarding_passes_p3_flight_id ON bookings.boarding_passes_p3 USING btree (flight_id);

CREATE INDEX idx_boarding_passes_p4_ticket_no ON bookings.boarding_passes_p4 (ticket_no);
CREATE INDEX idx_boarding_passes_p4_flight_id ON bookings.boarding_passes_p4 USING btree (flight_id);


```
```
-- Перенос данных из старой таблицы в новую секционированную таблицу
INSERT INTO bookings.boarding_passes_p (ticket_no, flight_id, boarding_no, seat_no)
SELECT ticket_no, flight_id, boarding_no, seat_no
FROM bookings.boarding_passes;


```

Результат тестирования показал, что секционирования не улучшило производительность, спустя несколько тестов подтвердилась гипотеза, что оно только ее ухудшило, вот пример результата:

```
-- секционирование таблиц
transaction type: bookings_bench34.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 162332
number of failed transactions: 0 (0.000%)
latency average = 55.435 ms
initial connection time = 162.261 ms
tps = 180.391067 (without initial connection time)
statement latencies in milliseconds and failures:
         0.017           0  \set book_ref random(100000, 999999)
         0.015           0  \set ticket_no random(1000000000000, 9999999999999)
         0.015           0  \set passenger_id random(1000, 9999)
         0.014           0  \set boarding_no random(1, 200)
         0.015           0  \set seat_no random(1000, 9999)
         0.014           0  \set new_seat_no random(1000, 9999)
         0.015           0  \set amount random(100, 10000) / 100.0
         0.015           0  \set new_amount random(100, 10000) / 100.0
         0.015           0  \set total_amount random(100, 10000) / 100.0
         0.014           0  \set flight_id random(1, 200000)
         0.734           0  BEGIN;
         0.849           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         0.727           0  END;
         0.585           0  BEGIN;
         0.813           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.653           0  END;
         0.564           0  BEGIN;
         0.986           0  INSERT INTO bookings.tickets_p (ticket_no, book_ref, passenger_id, passenger_name)
         0.777           0  END;
         0.590           0  BEGIN;
         0.922           0  SELECT * FROM bookings.tickets_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.694           0  END;
         0.576           0  BEGIN;
         1.467           0  INSERT INTO bookings.ticket_flights_p (ticket_no, flight_id, fare_conditions, amount)
         0.910           0  END;
         0.660           0  BEGIN;
         1.043           0  SELECT * FROM bookings.ticket_flights_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.749           0  END;
         0.610           0  BEGIN;
         1.463           0  INSERT INTO bookings.boarding_passes_p (ticket_no, flight_id, boarding_no, seat_no)
         0.924           0  END;
         0.671           0  BEGIN;
         1.046           0  SELECT * FROM bookings.boarding_passes_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.757           0  END;
         0.615           0  BEGIN;
         1.058           0  UPDATE bookings.boarding_passes_p SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight
         0.824           0  END;
         0.615           0  BEGIN;
         0.983           0  SELECT * FROM bookings.boarding_passes_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.725           0  END;
         0.021           0  \set old_amount :amount
         0.596           0  BEGIN;
         1.046           0  UPDATE bookings.ticket_flights_p SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :fl
         0.812           0  END;
         0.606           0  BEGIN;
         0.972           0  SELECT * FROM bookings.ticket_flights_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.721           0  END;
         0.591           0  BEGIN;
         1.017           0  UPDATE bookings.tickets_p SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.797           0  END;
         0.597           0  BEGIN;
         0.916           0  SELECT * FROM bookings.tickets_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.694           0  END;
         0.578           0  BEGIN;
         0.893           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         0.740           0  END;
         0.575           0  BEGIN;
         0.802           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.644           0  END;
         0.556           0  BEGIN;
         0.931           0  DELETE FROM bookings.boarding_passes_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.755           0  END;
         0.576           0  BEGIN;
         1.392           0  DELETE FROM bookings.ticket_flights_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.915           0  END;
         0.653           0  BEGIN;
         1.306           0  DELETE FROM bookings.tickets_p WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.910           0  END;
         0.657           0  BEGIN;
         1.584           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.965           0  END;




```
```
-- без секционирования
transaction type: bookings_bench32.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 900 s
number of transactions actually processed: 172241
number of failed transactions: 0 (0.000%)
latency average = 52.246 ms
initial connection time = 159.061 ms
tps = 191.402294 (without initial connection time)
statement latencies in milliseconds and failures:
         0.016           0  \set book_ref random(100000, 999999)
         0.014           0  \set ticket_no random(1000000000000, 9999999999999)
         0.014           0  \set passenger_id random(1000, 9999)
         0.014           0  \set boarding_no random(1, 200)
         0.014           0  \set seat_no random(1000, 9999)
         0.014           0  \set new_seat_no random(1000, 9999)
         0.014           0  \set amount random(100, 10000) / 100.0
         0.014           0  \set new_amount random(100, 10000) / 100.0
         0.014           0  \set total_amount random(100, 10000) / 100.0
         0.014           0  \set flight_id random(1, 200000)
         0.619           0  BEGIN;
         0.731           0  INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
         0.613           0  END;
         0.503           0  BEGIN;
         0.721           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.561           0  END;
         0.493           0  BEGIN;
         1.068           0  INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name)
         0.668           0  END;
         0.514           0  BEGIN;
         0.744           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.568           0  END;
         0.495           0  BEGIN;
         2.184           0  INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
         0.727           0  END;
         0.543           0  BEGIN;
         0.832           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.604           0  END;
         0.513           0  BEGIN;
         1.803           0  INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
         0.692           0  END;
         0.529           0  BEGIN;
         0.820           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.596           0  END;
         0.511           0  BEGIN;
         0.848           0  UPDATE bookings.boarding_passes SET seat_no = :new_seat_no::text WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_i
         0.660           0  END;
         0.512           0  BEGIN;
         0.791           0  SELECT * FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.584           0  END;
         0.021           0  \set old_amount :amount
         0.505           0  BEGIN;
         0.917           0  UPDATE bookings.ticket_flights SET amount = :new_amount WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flig
         0.679           0  END;
         0.518           0  BEGIN;
         0.797           0  SELECT * FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.589           0  END;
         0.506           0  BEGIN;
         0.774           0  UPDATE bookings.tickets SET passenger_name = 'Jane Doe' WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.635           0  END;
         0.503           0  BEGIN;
         0.732           0  SELECT * FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.564           0  END;
         0.492           0  BEGIN;
         0.795           0  UPDATE bookings.bookings SET total_amount = total_amount + (:old_amount - :new_amount) WHERE book_ref = LPAD(:book_ref::text, 1
         0.636           0  END;
         0.501           0  BEGIN;
         0.722           0  SELECT * FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.560           0  END;
         0.489           0  BEGIN;
         0.770           0  DELETE FROM bookings.boarding_passes WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.627           0  END;
         0.496           0  BEGIN;
         0.951           0  DELETE FROM bookings.ticket_flights WHERE ticket_no = LPAD(:ticket_no::text, 14, '0') AND flight_id = :flight_id;
         0.688           0  END;
         0.523           0  BEGIN;
         0.893           0  DELETE FROM bookings.tickets WHERE ticket_no = LPAD(:ticket_no::text, 14, '0');
         0.678           0  END;
         0.517           0  BEGIN;
         1.235           0  DELETE FROM bookings.bookings WHERE book_ref = LPAD(:book_ref::text, 11, '0');
         0.762           0  END;


```

14.  Идея для продолжения проекта: дополнить все 4 таблицы, участвующие в удалении строк, столбцами created_at, updated_at, deleted_at с типом данных timestamp и вместо DELETE  использовать UPDATE столбца deleted_at, который по умолчанию NULL. Также написать функцию, которая либо по запросу с бэеканда либо периолически чистила бы строки != NULL  в столбце deleted_at.

Полный список тестов (за исключением неудачных) будет прикреплен в этот же репозиторий (results_bench.txt).


   
        














