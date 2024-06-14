

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
yc-user@otus-vm:~$ 

work_mem не повлияло в лучшую сторону
wal_writer тоже не повлиял на ситуацию

CREATE INDEX idx_book_ref ON bookings.bookings (book_ref);
CREATE INDEX idx_ticket_no ON bookings.tickets (ticket_no);
```

Для корректной работы функции пришлось создать поседовательность, тк без нее процедура пыталась создать дубликаты: 
```
-- Найдите максимальное числовое значение book_ref
DO $$
DECLARE
    max_book_ref integer;
BEGIN
    SELECT COALESCE(MAX(CAST(book_ref AS integer)), 0) + 1 INTO max_book_ref FROM bookings.bookings WHERE book_ref ~ '^\d+$';
    
    -- Пересоздайте последовательность
    EXECUTE 'DROP SEQUENCE IF EXISTS bookings.book_ref_seq';
    EXECUTE format('CREATE SEQUENCE bookings.book_ref_seq START WITH %s INCREMENT BY 1 NO MINVALUE NO MAXVALUE CACHE 1', max_book_ref);
END;
$$;

```
