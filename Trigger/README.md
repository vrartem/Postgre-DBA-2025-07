	### Создаю таблицы и заполняю данными из условия ДЗ
	
```
psql (18.0 (Debian 18.0-1.pgdg13+3), server 17.6 (Debian 17.6-1.pgdg13+1))
Type "help" for help.

postgres=# DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;
NOTICE:  schema "pract_functions" does not exist, skipping
DROP SCHEMA
CREATE SCHEMA
postgres=# SET search_path = pract_functions, publ;
SET
postgres=# CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
CREATE TABLE
postgres=# INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
                (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
postgres=# CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
CREATE TABLE
INSERT 0 4
postgres=# SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

postgres=# CREATE TABLE good_sum_mart
(
        good_name   varchar(63) NOT NULL,
        sum_sale        numeric(16, 2)NOT NULL
);
CREATE TABLE
```

	### Создаю триггерную функцию
	
```
-- Создаем функцию для обновления витрины
CREATE OR REPLACE FUNCTION update_good_sum_mart()
RETURNS TRIGGER AS $$
BEGIN
	-- Обработка INSERT
	IF TG_OP = 'INSERT' THEN
        UPDATE good_sum_mart 
		SET sum_sale = sum_sale + (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);
        IF NOT FOUND THEN
            INSERT INTO good_sum_mart (good_name, sum_sale)
            SELECT
                good_name,
                good_price * NEW.sales_qty
            FROM goods
            WHERE goods_id = NEW.good_id;
        END IF;

        RETURN NEW;
	
	-- Обработка UPDATE
    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD.good_id <> NEW.good_id THEN
            -- Вычитаем
            UPDATE good_sum_mart
            SET sum_sale = sum_sale - (SELECT good_price FROM goods WHERE goods_id = OLD.good_id) * OLD.sales_qty
            WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);

            -- Добавляем
            UPDATE good_sum_mart
            SET sum_sale = sum_sale + (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * NEW.sales_qty
            WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);

            IF NOT FOUND THEN
                INSERT INTO good_sum_mart (good_name, sum_sale)
                SELECT good_name, COALESCE(good_price, 0) * NEW.sales_qty
                FROM goods
                WHERE goods_id = NEW.good_id;
            END IF;
        ELSE
            -- Изменилось только количество
            UPDATE good_sum_mart
            SET sum_sale = sum_sale + (SELECT good_price FROM goods WHERE goods_id = NEW.good_id) * (NEW.sales_qty - OLD.sales_qty)
            WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = NEW.good_id);
        END IF;

        RETURN NEW;

	-- Обработка DELETE
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE good_sum_mart
        SET sum_sale = sum_sale - (SELECT good_price FROM goods HERE goods_id = OLD.good_id ) * OLD.sales_qty
        WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id);
		-- Если 0 или меньше
        DELETE FROM good_sum_mart WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = OLD.good_id) AND sum_sale <= 0;
        RETURN OLD;
    END IF;

    RETURN NULL;

END;
$$ LANGUAGE plpgsql;
```

	### Создаю триггер
	
```
CREATE TRIGGER sales_trigger
    AFTER INSERT OR UPDATE OR DELETE ON pract_functions.sales
    FOR EACH ROW EXECUTE FUNCTION update_good_sum_mart();
```

	### Проверяю вставку
```
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10);
INSERT INTO sales (good_id, sales_qty) VALUES (2, 1);

postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |         5.00
 Автомобиль Ferrari FXX K | 185000000.01
(2 rows)
```

	### Проверяю UPDATE
	
```
postgres=# UPDATE sales SET sales_qty = 30 WHERE sales_id = 2;
UPDATE 1
postgres=# Select * from sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2025-12-01 12:44:57.069665+00 |        10
        3 |       1 | 2025-12-01 12:44:57.069665+00 |       120
        4 |       2 | 2025-12-01 12:44:57.069665+00 |         1
        5 |       1 | 2025-12-01 13:30:59.213417+00 |        10
        6 |       2 | 2025-12-01 13:31:47.329222+00 |         1
        2 |       1 | 2025-12-01 12:44:57.069665+00 |        30
(6 rows)

postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        19.50
```

	### Проверяю DELETE

```
postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        60.00
(2 rows)

postgres=# Select * from sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2025-12-01 12:44:57.069665+00 |        10
        4 |       2 | 2025-12-01 12:44:57.069665+00 |         1
        5 |       1 | 2025-12-01 13:30:59.213417+00 |        10
        6 |       2 | 2025-12-01 13:31:47.329222+00 |         1
        2 |       1 | 2025-12-01 12:44:57.069665+00 |        30
        7 |       1 | 2025-12-01 13:42:09.979394+00 |       120
(6 rows)

postgres=# DELETE FROM sales WHERE sales_id = 1;
DELETE 1
postgres=# Select * from sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        4 |       2 | 2025-12-01 12:44:57.069665+00 |         1
        5 |       1 | 2025-12-01 13:30:59.213417+00 |        10
        6 |       2 | 2025-12-01 13:31:47.329222+00 |         1
        2 |       1 | 2025-12-01 12:44:57.069665+00 |        30
        7 |       1 | 2025-12-01 13:42:09.979394+00 |       120
(5 rows)

postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        55.00
(2 rows)
```

	### Преимущества схемы
	
- Цены меняются, при такой схеме мы берем фактические значения цен на тот момент времени.
- Данные уже готовы, вывод отчета будет работать быстро.