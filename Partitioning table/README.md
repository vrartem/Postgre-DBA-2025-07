### 1. Выбираю таблицу для секционирования bookings.bookings из db demo

Хотя база всего за 3 месяца в таблце уже 1292893 записей.
book_date хорошо подходит в качестве ключа секционирования.
Запросы по диапазону дат будут затрагивать несколько партиций вместо миллионов строк всей таблицы.

### 2. Определяю тип секционирования
Буду секциоанировать таблицу по book_date с типом секционирования range по месяцам.

Результат:



### Переношу данные в новую таблицу

 ```
demo=# INSERT INTO bookings.bookings_new SELECT * FROM bookings.bookings;
INSERT 0 1292893
```

### Проверяю распределение
```
-- 5. Проверяем распределение
demo=# select date_part('year',book_date),date_part('month',book_date),count(*)
from bookings.bookings_new
group by date_part('year',book_date),date_part('month',book_date)
order by date_part('year',book_date),date_part('month',book_date);
 date_part | date_part | count
-----------+-----------+--------
      2025 |         9 | 448064
      2025 |        10 | 434159
      2025 |        11 | 410670
(3 rows)
```

### Тестирование и оптимизация

-  Для обычной

```
 EXPLAIN ANALYZE SELECT COUNT(*) FROM bookings.bookings WHERE book_date = '2025-10-11';

 Aggregate  (cost=15973.92..15973.93 rows=1 width=8) (actual time=18.054..19.977 rows=1 loops=1)
   ->  Gather  (cost=1000.00..15973.92 rows=1 width=0) (actual time=18.052..19.974 rows=0 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Parallel Seq Scan on bookings  (cost=0.00..14973.82 rows=1 width=0) (actual time=15.766..15.767 rows=0 loops=3)
               Filter: (book_date = '2025-10-11 00:00:00+00'::timestamp with time zone)
               Rows Removed by Filter: 430964
 Planning Time: 0.044 ms
 Execution Time: 19.992 ms
```
 - С секционированием
 ```
 EXPLAIN ANALYZE SELECT COUNT(*) FROM bookings.bookings_new WHERE book_date = '2025-10-11';
 Aggregate  (cost=6958.45..6958.46 rows=1 width=8) (actual time=9.198..11.497 rows=1 loops=1)
   ->  Gather  (cost=1000.00..6958.45 rows=1 width=0) (actual time=9.195..11.494 rows=0 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Parallel Seq Scan on bookings_new_202510 bookings_new  (cost=0.00..5958.35 rows=1 width=0) (actual time=7.524..7.524 rows=0 loops=2)
               Filter: (book_date = '2025-10-11 00:00:00'::timestamp without time zone)
               Rows Removed by Filter: 217080
 Planning Time: 0.069 ms
 Execution Time: 11.516 ms
```

Секционирование сработало.
Второй запрос в 2.3 раза дешевле по оценке планировщика.

 - Дополнительно попробовал добавить индекс на одну партицию

```
CREATE INDEX idx_bookings_new_book_date ON bookings.bookings_new_202510 (book_date);

EXPLAIN ANALYZE SELECT COUNT(*) FROM bookings.bookings_new WHERE book_date = '2025-10-11';

 Aggregate  (cost=4.44..4.45 rows=1 width=8) (actual time=0.048..0.048 rows=1 loops=1)
   ->  Index Only Scan using idx_bookings_new_book_date on bookings_new_202510 bookings_new  (cost=0.42..4.44 rows=1 width=0) (actual time=0.035..0.036 rows=0 loops=1)
         Index Cond: (book_date = '2025-10-11 00:00:00'::timestamp without time zone)
         Heap Fetches: 0
 Planning Time: 0.421 ms
 Execution Time: 0.178 ms
```

Это хороший результат.
Теперь выборка, которая затронет две партиции

```
 EXPLAIN ANALYZE SELECT COUNT(*) FROM bookings.bookings_new WHERE book_date > '2025-10-13' and book_date < '2025-11-13';
 Finalize Aggregate  (cost=15112.99..15113.00 rows=1 width=8) (actual time=45.633..52.330 rows=1 loops=1)
   ->  Gather  (cost=15112.78..15112.99 rows=2 width=8) (actual time=45.538..52.325 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=14112.78..14112.79 rows=1 width=8) (actual time=42.991..42.992 rows=1 loops=3)
               ->  Parallel Append  (cost=0.00..13666.47 rows=178524 width=0) (actual time=0.041..38.924 rows=142816 loops=3)
                     ->  Parallel Index Only Scan using idx_bookings_new_book_date on bookings_new_202510 bookings_new_1  (cost=0.42..6534.29 rows=108172 width=0) (actual time=0.102..12.429 rows=86881 loops=3)
                           Index Cond: ((book_date > '2025-10-13 00:00:00'::timestamp without time zone) AND (book_date < '2025-11-13 00:00:00'::timestamp without time zone))
                           Heap Fetches: 0
                     ->  Parallel Seq Scan on bookings_new_202511 bookings_new_2  (cost=0.00..6239.56 rows=99321 width=0) (actual time=0.033..29.971 rows=83902 loops=2)
                           Filter: ((book_date > '2025-10-13 00:00:00'::timestamp without time zone) AND (book_date < '2025-11-13 00:00:00'::timestamp without time zone))
                           Rows Removed by Filter: 121432
 Planning Time: 0.590 ms
 Execution Time: 52.391 ms
```

- Создал индекс для таблицы без партиций и решел сравнить

```
CREATE INDEX idx_bookings_book_date ON bookings.bookings (book_date);

EXPLAIN ANALYZE SELECT COUNT(*) FROM bookings.bookings WHERE book_date > '2025-10-13' and book_date < '2025-11-13';

 Finalize Aggregate  (cost=12262.11..12262.12 rows=1 width=8) (actual time=38.148..40.324 rows=1 loops=1)
   ->  Gather  (cost=12261.89..12262.10 rows=2 width=8) (actual time=38.057..40.318 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=11261.89..11261.90 rows=1 width=8) (actual time=35.747..35.748 rows=1 loops=3)
               ->  Parallel Index Only Scan using idx_bookings_book_date on bookings  (cost=0.43..10814.24 rows=179063 width=0) (actual time=0.070..31.625 rows=142816 loops=3)
                     Index Cond: ((book_date > '2025-10-13 00:00:00+00'::timestamp with time zone) AND (book_date < '2025-11-13 00:00:00+00'::timestamp with time zone))
                     Heap Fetches: 0
 Planning Time: 0.083 ms
 Execution Time: 40.343 ms
```

- Результат для bookings.bookings получился лучше

Создаю индекс для bookings.bookings_new_202511

```
CREATE INDEX idx_bookings_new_202511_book_date ON bookings.bookings_new_202511 (book_date);

EXPLAIN ANALYZE SELECT COUNT(*) FROM bookings.bookings_new WHERE book_date > '2025-10-13' and book_date < '2025-11-13';

 Finalize Aggregate  (cost=13126.53..13126.54 rows=1 width=8) (actual time=25.506..27.672 rows=1 loops=1)
   ->  Gather  (cost=13126.32..13126.53 rows=2 width=8) (actual time=25.407..27.668 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=12126.32..12126.33 rows=1 width=8) (actual time=22.855..22.856 rows=1 loops=3)
               ->  Parallel Append  (cost=0.42..11679.97 rows=178541 width=0) (actual time=0.071..18.807 rows=142816 loops=3)
                     ->  Parallel Index Only Scan using idx_bookings_new_book_date on bookings_new_202510 bookings_new_1  (cost=0.42..6534.29 rows=108172 width=0) (actual time=0.079..7.828 rows=86881 loops=3)
                           Index Cond: ((book_date > '2025-10-13 00:00:00'::timestamp without time zone) AND (book_date < '2025-11-13 00:00:00'::timestamp without time zone))
                           Heap Fetches: 0
                     ->  Parallel Index Only Scan using idx_bookings_new_202511_book_date on bookings_new_202511 bookings_new_2  (cost=0.42..4252.97 rows=70369 width=0) (actual time=0.025..6.811 rows=83902 loops=2)
                           Index Cond: ((book_date > '2025-10-13 00:00:00'::timestamp without time zone) AND (book_date < '2025-11-13 00:00:00'::timestamp without time zone))
                           Heap Fetches: 0
 Planning Time: 0.309 ms
 Execution Time: 27.715 ms
```

Оценочная стоимость ниже у bookings.bookings, но реальное время выполнения выше.
Интересные получились наблюдения.
Результаты, возможно, были другими если я взял демо базу большего размера.