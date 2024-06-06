```
-- Создание процедуры для заполнения данных
DO $$
DECLARE
    aircraft_codes text[] := array['A321', 'B737', 'C123'];
    airports text[] := array['JFK', 'LAX', 'SFO', 'ORD', 'ATL'];
    fare_conditions text[] := array['Economy', 'Comfort', 'Business'];
    start_date timestamp := '2023-01-01 00:00:00';
    end_date timestamp := '2024-01-01 00:00:00';
    total_records int := 1000000; -- Укажите общее количество записей, чтобы заполнить около 30 ГБ данных
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
        INSERT INTO bookings.aircrafts_data (aircraft_code, model, range)
        VALUES (aircraft_codes[i], jsonb_build_object('en', 'Model ' || aircraft_codes[i]), 5000 + i * 1000);
    END LOOP;

    -- Заполнение таблицы airports_data
    FOR i IN 1..array_length(airports, 1) LOOP
        INSERT INTO bookings.airports_data (airport_code, airport_name, city, coordinates, timezone)
        VALUES (airports[i], jsonb_build_object('en', 'Airport ' || airports[i]), jsonb_build_object('en', 'City ' || airports[i]), point(40.0 + i, -74.0 + i), 'UTC');
    END LOOP;

    -- Заполнение таблицы bookings и связанной с ней tickets
    FOR i IN 1..total_records LOOP
        book_ref := lpad(i::text, 6, '0');
        book_date := start_date + random() * (end_date - start_date);

        INSERT INTO bookings.bookings (book_ref, book_date, total_amount)
        VALUES (book_ref, book_date, round(random() * 1000, 2));

        ticket_no := lpad((i * 10)::text, 13, '0');

        INSERT INTO bookings.tickets (ticket_no, book_ref, passenger_id, passenger_name, contact_data)
        VALUES (ticket_no, book_ref, 'ID' || i, 'Passenger ' || i, jsonb_build_object('phone', '1234567890'));

        FOR j IN 1..5 LOOP
            flight_id := nextval('bookings.flights_flight_id_seq');
            scheduled_departure := start_date + random() * (end_date - start_date);
            scheduled_arrival := scheduled_departure + interval '1 hour' * (1 + random() * 5);

            INSERT INTO bookings.flights (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code)
            VALUES (flight_id, lpad(flight_id::text, 6, '0'), scheduled_departure, scheduled_arrival, airports[j % array_length(airports, 1) + 1], airports[(j + 1) % array_length(airports, 1) + 1], 'Scheduled', aircraft_codes[j % array_length(aircraft_codes, 1) + 1]);

            INSERT INTO bookings.ticket_flights (ticket_no, flight_id, fare_conditions, amount)
            VALUES (ticket_no, flight_id, fare_conditions[j % array_length(fare_conditions, 1) + 1], round(random() * 500, 2));
```

            INSERT INTO bookings.boarding_passes (ticket_no, flight_id, boarding_no, seat_no)
            VALUES (ticket_no, flight_id, j, 'A' || j);
        END LOOP;
    END LOOP;
END;
$$ LANGUAGE plpgsql; '''
