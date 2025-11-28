### Тестовая таблица
```
CREATE TABLE bookings (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    hotel_id INT NOT NULL,
    guest_name TEXT NOT NULL,
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    room_type TEXT NOT NULL,
    total_price NUMERIC(8,2) NOT NULL,
    is_confirmed BOOLEAN NOT NULL DEFAULT TRUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Заполняем данными

```
postgres=# INSERT INTO bookings (
    hotel_id, guest_name, check_in, check_out, room_type, total_price, is_confirmed, description
)
SELECT
    FLOOR(RANDOM() * 10 + 1),
    'Гость ' || LPAD(FLOOR(RANDOM()*1000)::TEXT, 4, '0'),
    check_in_val,
    check_in_val + (nights * INTERVAL '1 day'),
    CASE FLOOR(RANDOM() * 3)
        WHEN 0 THEN 'Standard'
        WHEN 1 THEN 'Deluxe'
        ELSE 'Suite'
    END,
    5000 + (RANDOM() * 15000),
    RANDOM() < 0.8,
    CASE
        WHEN RANDOM() < 0.3 THEN 'Ранний заезд'
        WHEN RANDOM() < 0.6 THEN 'Поздний выезд'
        WHEN RANDOM() < 0.8 THEN 'Нужен трансфер из аэропорта'
        ELSE NULL
    END
FROM
    (
        SELECT
            CURRENT_DATE + (FLOOR(RANDOM() * 365) * INTERVAL '1 day') AS check_in_val,
            FLOOR(RANDOM() * 7 + 1) AS nights
        FROM generate_series(1, 1000000)
    ) AS generated_dates;
```

### Пробую делат запрос без индексов

```
postgres=# EXPLAIN ANALYZE SELECT * FROM bookings
WHERE hotel_id = 5  AND check_in BETWEEN '2026-01-01' AND '2026-01-05';
                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..24287.44 rows=8700 width=93) (actual time=0.447..56.080 rows=1354 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings  (cost=0.00..22417.44 rows=3625 width=93) (actual time=0.039..33.674 rows=451 loops=3)
         Filter: ((check_in >= '2026-01-01'::date) AND (check_in <= '2026-01-05'::date) AND (hotel_id = 5))
         Rows Removed by Filter: 332882
 Planning Time: 0.127 ms
 Execution Time: 56.165 ms
 ```
 Просто выполняется параллельное последовательное сканирование таблицы.

 ### Создаю индекс по двум столбцам и проверяю
 
 ```
 CREATE INDEX idx_hotel_checkin ON bookings (hotel_id, check_in);

 postgres=# EXPLAIN ANALYZE SELECT * FROM bookings
WHERE hotel_id = 5  AND check_in BETWEEN '2026-01-01' AND '2026-01-05';
                                                           QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings  (cost=23.97..3752.46 rows=1219 width=91) (actual time=0.289..1.747 rows=1354 loops=1)
   Recheck Cond: ((hotel_id = 5) AND (check_in >= '2026-01-01'::date) AND (check_in <= '2026-01-05'::date))
   Heap Blocks: exact=1299
   ->  Bitmap Index Scan on idx_hotel_checkin  (cost=0.00..23.66 rows=1219 width=0) (actual time=0.154..0.154 rows=1354 loops=1)
         Index Cond: ((hotel_id = 5) AND (check_in >= '2026-01-01'::date) AND (check_in <= '2026-01-05'::date))
 Planning Time: 0.259 ms
 Execution Time: 4.094 ms
 ```
Теперь запрос использует индексный доступ, Index Scan - считывает указатели строк из индекса

При выборке по одному из полей индекс срабатывать не будет

```
postgres=#  EXPLAIN ANALYZE SELECT * FROM bookings
WHERE check_in BETWEEN '2026-01-01' AND '2026-01-05';
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..23543.00 rows=12330 width=91) (actual time=0.856..92.007 rows=13734 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings  (cost=0.00..21310.00 rows=5138 width=91) (actual time=0.018..15.658 rows=4578 loops=3)
         Filter: ((check_in >= '2026-01-01'::date) AND (check_in <= '2026-01-05'::date))
         Rows Removed by Filter: 328755
 Planning Time: 0.116 ms
 Execution Time: 92.378 ms
```

 
 ### Частичный индекс для неподтверждённых бронирований

``` 
postgres=# CREATE INDEX idx_confirmed ON bookings (check_in) WHERE is_confirmed = FALSE;
CREATE INDEX
postgres=# explain analyze
SELECT * FROM bookings WHERE is_confirmed = false AND check_in >= CURRENT_DATE;
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings  (cost=2269.24..20319.46 rows=199348 width=91) (actual time=6.953..33.060 rows=199839 loops=1)
   Recheck Cond: ((check_in >= CURRENT_DATE) AND (NOT is_confirmed))
   Heap Blocks: exact=15060
   ->  Bitmap Index Scan on idx_confirmed  (cost=0.00..2219.41 rows=199348 width=0) (actual time=5.468..5.468 rows=199839 loops=1)
         Index Cond: (check_in >= CURRENT_DATE)
 Planning Time: 0.179 ms
 Execution Time: 37.757 ms
```
Заметил, если is_confirmed = false встречается редко, индекс работает, если часто - не будет применяться.

```
postgres=# CREATE INDEX idx_confirmed_true ON bookings (check_in) WHERE is_confirmed = TRUE;
CREATE INDEX
postgres=# explain analyze
SELECT * FROM bookings
WHERE is_confirmed = TRUE AND check_in >= CURRENT_DATE;
                                                    QUERY PLAN
------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings  (cost=0.00..30060.00 rows=800556 width=91) (actual time=0.007..64.428 rows=800161 loops=1)
   Filter: (is_confirmed AND (check_in >= CURRENT_DATE))
   Rows Removed by Filter: 199839
 Planning Time: 0.294 ms
 Execution Time: 83.057 ms
```
Получается высокая доля отбираемых строк и выборка через индекс может быть медленнее из‑за накладных расходов на чтение индексных страниц.
 
 ### Индекс для полнотекстового поиска

``` 
CREATE INDEX idx_desc ON bookings USING gin(to_tsvector('russian', description));

postgres=# EXPLAIN ANALYZE
SELECT * FROM bookings WHERE to_tsvector('russian', description) @@ to_tsquery('russian', 'заезд');
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings  (cost=191.52..11793.40 rows=5000 width=91) (actual time=17.798..49.897 rows=299432 loops=1)
   Recheck Cond: (to_tsvector('russian'::regconfig, description) @@ '''заезд'''::tsquery)
   Heap Blocks: exact=15060
   ->  Bitmap Index Scan on idx_desc  (cost=0.00..190.27 rows=5000 width=0) (actual time=16.271..16.271 rows=299432 loops=1)
         Index Cond: (to_tsvector('russian'::regconfig, description) @@ '''заезд'''::tsquery)
 Planning Time: 5.263 ms
 Execution Time: 56.864 ms
 ```
Индекс idx_desc работает корректно.
 
