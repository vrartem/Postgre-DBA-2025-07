## Конфигурация виртуальной машины
- **CPU**: 4 ядра  
- **Оперативная память**: 6ГБ   
- **Диск**: 120ГБ SSD

### Используемое ПО
* Ubuntu 24.04.3
* PostgreSQL 16.10 (Ubuntu 16.10-0ubuntu0.24.04.1)
* Расширение PostGIS 3.4.2
* shp2pgsql
* DBeaver 25.2.2
* QGIS


## Установка кластера PostgreSQL

### Установка пакетов, включение и запуск службы

```
sudo apt install postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```
### Проверка статуса 
```sudo systemctl status postgresql```

![](https://github.com/vrartem/Postgre-DBA-2025-07/blob/main/Project/1.png) 

### Установка пароля для пользователя postgres
```
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD '';
```
### Запуск нагрузочного тестирования на дефолтных настройках
```
sudo -u postgres  pgbench -i -s 10 geobase
sudo -u postgres  pgbench -c 10 -T 60 geobase

```
Результат:
```
sudo -u postgres  pgbench -c 10 -T 60 geobase
pgbench (16.10 (Ubuntu 16.10-0ubuntu0.24.04.1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 109010
number of failed transactions: 0 (0.000%)
latency average = 5.497 ms
initial connection time = 94.996 ms
tps = 1819.151508 (without initial connection time)
```
### Настройка кластера
Использовал https://www.pgconfig.org/ конфигуратор

Основные изменения в /etc/postgresql/16/main/postgresql.conf:

```
listen_addresses = '*'
max_connections = 100
ssl = on
shared_buffers = 2GB
work_mem = 26MB
maintenance_work_mem = 307MB
effective_cache_size = 5GB
max_wal_size = 3GB
min_wal_size = 2GB
```
изменения в /etc/postgresql/16/main/pg_hba.conf:
```
host    geobase         geouser         0.0.0.0/0               scram-sha-256
```
Повторный запуск теста
```
root@localhost:~# sudo -u postgres  pgbench -c 10 -T 60 geobase
pgbench (16.10 (Ubuntu 16.10-0ubuntu0.24.04.1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 110424
number of failed transactions: 0 (0.000%)
latency average = 5.425 ms
initial connection time = 111.318 ms
tps = 1843.255418 (without initial connection time)
```
## Организация структуры базы данных и прав доступа

### Создание новой базы данных, схемы, пользователя, добавление доступов

```
sudo -u postgres psql
CREATE DATABASE GeoBase;
CREATE USER geouser WITH PASSWORD '';
GRANT ALL PRIVILEGES ON DATABASE "geobase" TO geouser;
\c geobase
CREATE SCHEMA geo;
GRANT USAGE ON SCHEMA geo TO geouser;
ALTER DEFAULT PRIVILEGES IN SCHEMA geo GRANT SELECT ON TABLES TO geouser; - права по умолчанию для новых таблиц
```
### Проверка подключения

```
psql -U geouser -d geobase -h 127.0.0.1 -p 5432
geobase=> \conninfo
You are connected to database "geobase" as user "geouser" on host "127.0.0.1" at port "5432".
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
```

### Устанавливаю расширение postgis

```
apt list | grep postgis
пакеты есть, устанавливаю
sudo apt install postgresql-16-postgis-3 postgis
root@localhost:~# psql -U geouser -d geobase -h 127.0.0.1 -p 5432
geobase=> CREATE EXTENSION postgis;
ERROR:  permission denied to create extension "postgis"
HINT:  Must be superuser to create this extension.
```
установить расширение под geouser запрещено, захожу под postgres и повторяю
```
geobase=> CREATE EXTENSION postgis;
SELECT PostGIS_full_version();
POSTGIS="3.4.2 c19ce56" [EXTENSION] PGSQL="160" GEOS="3.12 …
```

### Порт 5432 закрыт, необходимо его открыть

Открываю:
```
sudo iptables -A INPUT -p tcp --dport 5432 -j ACCEPT
```
Проверяю:
```
sudo iptables -L INPUT -v -n | grep 5432
692K  163M ACCEPT     6    --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:5432
```
Сохраняю правило
```
sudo netfilter-persistent save
```

## Создание sftp папки на сервере для подключения

```
sudo groupadd sftpusers
sudo useradd -m -G sftpusers sftpuser
sudo passwd sftpuser
sudo mkdir -p /home/sftpuser
sudo chown root:root /home/sftpuser
sudo chmod 755 /home/sftpuser
sudo mkdir /home/sftpuser/uploads
sudo chown sftpuser:sftpusers /home/sftpuser/uploads
sudo chmod 755 /home/sftpuser/uploads
sudo systemctl restart ssh
```

## Скрипт для выполнения импорта данных в базу из архива и спользованием shp2pgsql

```
root@localhost:/home/sftpuser/uploads# cat /home/sftpuser/import_shp_new.sh
#!/bin/bash

# Параметры
UPLOADS_DIR="/home/sftpuser/uploads"
TEMP_DIR="/tmp/shp_extract"
DB_NAME="geobase"
DB_USER="postgres"
LOG_FILE="/var/log/shp2pgsql_import.log"
PROCESSED_MARKER=".processed"  # маркер обработанных архивов


# Создаём временные директории
mkdir -p "$TEMP_DIR"


# Логируем начало
echo "=== Запуск импорта SHP ($(date)) ===" | tee -a "$LOG_FILE"

# Перебираем все ZIP‑архивы в директории uploads
for ARCHIVE in "$UPLOADS_DIR"/*.zip; do
    # Получаем имя файла без пути и расширения
    ARCHIVE_NAME=$(basename "$ARCHIVE" .zip)

    # Проверяем, обработан ли архив (есть ли маркер)
    if [ -f "$ARCHIVE$PROCESSED_MARKER" ]; then
        echo "Пропускаем уже обработанный архив: $ARCHIVE_NAME" | tee -a "$LOG_FILE"
        continue
    fi

    echo "Обрабатываем архив: $ARCHIVE_NAME" | tee -a "$LOG_FILE"

    # Распаковываем во временную директорию
    unzip -o "$ARCHIVE" -d "$TEMP_DIR/$ARCHIVE_NAME" | tee -a "$LOG_FILE"


    # Перебираем все SHP‑файлы в распакованной папке
    for SHP_FILE in "$TEMP_DIR/$ARCHIVE_NAME"/*.shp; do
        if [ -f "$SHP_FILE" ]; then
            FILENAME=$(basename "$SHP_FILE" .shp)
            echo "  Импорт $FILENAME в таблицу $FILENAME..." | tee -a "$LOG_FILE"

            # Выполняем импорт через shp2pgsql → psql
            shp2pgsql -I -s 4326 "$SHP_FILE" "geo.$ARCHIVE_NAME" \
                |PGPASSWORD="" psql -U "$DB_USER" -d "$DB_NAME" -h localhost -p 5432 2>&1 \
                | tee -a "$LOG_FILE"


            # Проверяем статус
            if [ $? -eq 0 ]; then
                echo "  Успешно: $FILENAME" | tee -a "$LOG_FILE"
            else
                echo "  Ошибка: $FILENAME" | tee -a "$LOG_FILE"
            fi
        fi
    done

    # Помечаем архив как обработанный
    touch "$ARCHIVE$PROCESSED_MARKER"
    echo "Архив $ARCHIVE_NAME обработан и помечен" | tee -a "$LOG_FILE"

    # Очищаем временную папку для этого архива
    rm -rf "$TEMP_DIR/$ARCHIVE_NAME"
done

echo "Импорт завершён. Лог: $LOG_FILE" | tee -a "$LOG_FILE"
```

## Создание наблюдателя за папкой

Будет запускаться скрипт import_shp_new.sh при добавлении архива в папку

Установка:

```
sudo apt install inotify-tools
```
Логика наблюдателя:

```
nano /usr/local/bin/watch_upload.sh

#!/bin/bash

WATCH_DIR="/home/sftpuser/uploads"
IMPORT_SCRIPT="/home/sftpuser/import_shp_new.sh"
LOG_FILE="/var/log/upload_watcher.log"

echo "Наблюдение за $WATCH_DIR запущено $(date)" | tee -a "$LOG_FILE"

inotifywait -m \
  -e create \
  --format '%e %f' "$WATCH_DIR" | while read EVENT FILENAME; do


  echo "Событие: $EVENT $FILENAME ($(date))" | tee -a "$LOG_FILE"

  # Обрабатываем только CREATE для ZIP‑файлов без .filepart
  if [[ "$EVENT! == "CREATE! ]] && \
     [[ "$FILENAME! == *.zip ]] && \
     [[ "$FILENAME! != *.filepart ]]; then

    echo "→ Запускаем импорт: $FILENAME" | tee -a "$LOG_FILE"
    sudo "$IMPORT_SCRIPT" "$WATCH_DIR/$FILENAME" >> "$LOG_FILE" 2>&1
  else
    echo "→ Пропускаем: $EVENT $FILENAME" | tee -a "$LOG_FILE"
  fi
done
```

### Как только новый zip-архив будет загружен на сервер, автоматически запустится импорт в базу данных и спользованием shp2pgsql где:
- -I — создаёт индекс GiST по колонке геометрии (ускоряет пространственные запросы).
- -s 4326 — задаёт SRID = 4326 (система координат WGS 84, долгота/широта).
- "$SHP_FILE" — путь к входному шейп‑файлу.
- "geo.$ARCHIVE_NAME" — имя целевой таблицы в схеме geo
- PGPASSWORD="" psql -U "$DB_USER" -d "$DB_NAME" -h localhost -p 5432 - Выполняет сгенерированный SQL в PostgreSQL под пользователем postgres
- tee -a "$LOG_FILE" -  вывод в лог файл $LOG_FILE

### Операция не атомарная, возможно можно улучшить через промежуточный файл sql и выполнение его в транзакции

### Пример построения периметра учатска

```
WITH POINTS AS (
SELECT (ST_DumpPoints(geom)).geom AS pt -- разбиваем геометрию на точки
FROM field_1 
WHERE geom IS NOT NULL)

SELECT ST_ConcaveHull(ST_Collect(pt), 0.1) AS concave_polygon 
FROM POINTS
```

### Создание слоев в QGIS для построения карты урожайности

```
SELECT * FROM "geo"."field_1" where rate < 0.24 -- минимальная
SELECT * FROM "geo"."field_1" where rate >= 0.24 and rate <= 0.4 -- средняя
SELECT * FROM "geo"."field_1" where rate > 0.4 -- максимальная
```

### Подсчет площади

```
SELECT ST_Area(ST_ConcaveHull(ST_Collect(geom), 0.5)::geography)
FROM (
	SELECT (ST_DumpPoints(geom)).geom
	FROM field_1
	WHERE geom IS NOT NULL
);

Результат: 455 280
 0.5 - параметр «тугости»
::geography — приведение геометрии к типу GEOGRAPHY для точных расчетов площади

```

### EXPLAIN ANALYZE

```
Aggregate  (cost=259832741.66..259832766.68 rows=1 width=8) (actual time=2871.860..2871.879 rows=1 loops=1)
  ->  Result  (cost=0.00..257265741.04 rows=20536000 width=32) (actual time=259.431..492.618 rows=230526 loops=1)
        ->  ProjectSet  (cost=0.00..360381.04 rows=20536000 width=32) (actual time=259.396..448.650 rows=230526 loops=1)
              ->  Seq Scan on field_1  (cost=0.00..898.36 rows=20536 width=227) (actual time=259.035..274.095 rows=20536 loops=1)
                    Filter: (geom IS NOT NULL)
Planning Time: 0.679 ms
JIT:
  Functions: 7
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 1.769 ms, Inlining 51.245 ms, Optimization 150.771 ms, Emission 57.193 ms, Total 260.979 ms
Execution Time: 2874.621 ms
```
Пространственный индекс не был использован, выбираются все записи и выполняются преобразования

Пространственный индекс ускоряет запросы с операторами и функциями
- ST_Contains, ST_Intersects, ST_Within — проверка вложенности/пересечения;

- ST_DWithin — поиск объектов в заданном радиусе;

- ST_Distance — расчёт расстояний (при определённых условиях);

- ST_NearestNeighbor (SDO_NN) — поиск ближайших объектов;

- ST_Envelope, ST_Area, ST_Length и др. — если они используются в условиях

### Подсчет площади пересечений

```
select
	SUM(ST_Area(ST_Intersection(a.geom, b.geom)::geography)) as total_intersection
from field_1 a
join field_1 b on ST_Intersects(a.geom, b.geom)
where a.gid + 1 < b.gid
and ST_Area(ST_Intersection(a.geom, b.geom)::geography) > 6
and ST_IsValid(a.geom) and ST_IsValid(b.geom)
```

### EXPLAIN ANALYZE

```
EXPLAIN ANALYZE
select
SUM(ST_Area(ST_Intersection(a.geom, b.geom)::geography)) AS total_intersection_area
from field_1 a
join field_1 b on ST_Intersects(a.geom, b.geom)
where a.gid + 1 < b.gid
and ST_Area(ST_Intersection(a.geom, b.geom)::geography) > 6
and ST_IsValid(a.geom)
AND ST_IsValid(b.geom)
Aggregate (cost=969071.95..969071.96 rows=1 width=8) (actual time=5531.897..5531.913 rows=1 loops=1)
-> Nested Loop (cost=0.28..947617.66 rows=858 width=454) (actual time=684.423..5454.399 rows=896 loops=1)
-> Seq Scan on field_1 a (cost=0.00..257598.36 rows=6845 width=231) (actual time=662.423..1010.792 rows=19404 loops=1)
Filter: st_isvalid(geom)
Rows Removed by Filter: 1132
-> Index Scan using field_1_geom_idx on field_1 b (cost=0.28..100.80 rows=1 width=231) (actual time=0.221..0.227 rows=0 loops=19404)
Index Cond: (geom && a.geom)
Filter: (((a.gid + 1) < gid) AND st_isvalid(geom) AND st_intersects(a.geom, geom) AND (st_area((st_intersection(a.geom, geom, '-1'::double precision))::geography, true) > '6'::double precision))
Rows Removed by Filter: 7
Planning Time: 36.674 ms
JIT:
Functions: 12
Options: Inlining true, Optimization true, Expressions true, Deforming true
Timing: Generation 3.176 ms, Inlining 351.308 ms, Optimization 196.677 ms, Emission 114.199 ms, Total 665.360 ms
Execution Time: 5651.158 ms
```
Видим использование Index Scan field_1_geom_idx по geom.

Запрос отработал за приемлимое время.

### Если удалить индекс по geom

```
Aggregate  (cost=1758616821.18..1758616821.19 rows=1 width=8) (actual time=143967.359..143967.375 rows=1 loops=1)
  ->  Nested Loop  (cost=0.00..1758595366.89 rows=858 width=454) (actual time=717.974..143885.701 rows=896 loops=1)
        Join Filter: (((a.gid + 1) < b.gid) AND st_intersects(a.geom, b.geom) AND (st_area((st_intersection(a.geom, b.geom, '-1'::double precision))::geography, true) > '6'::double precision))
        Rows Removed by Join Filter: 376514320
        ->  Seq Scan on field_1 a  (cost=0.00..257598.36 rows=6845 width=231) (actual time=412.894..1759.190 rows=19404 loops=1)
              Filter: st_isvalid(geom)
              Rows Removed by Filter: 1132
        ->  Materialize  (cost=0.00..257632.58 rows=6845 width=231) (actual time=0.001..2.241 rows=19404 loops=19404)
              ->  Seq Scan on field_1 b  (cost=0.00..257598.36 rows=6845 width=231) (actual time=0.135..250.811 rows=19404 loops=1)
                    Filter: st_isvalid(geom)
                    Rows Removed by Filter: 1132
Planning Time: 2.431 ms
JIT:
  Functions: 14
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 5.207 ms, Inlining 49.201 ms, Optimization 251.817 ms, Emission 112.207 ms, Total 418.432 ms
Execution Time: 143975.786 ms
```

#### Примерно в 25.5 раз падает эффективность


