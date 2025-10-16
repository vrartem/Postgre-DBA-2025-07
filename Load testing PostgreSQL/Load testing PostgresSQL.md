## Нагрузочное тестирование и тюнинг PostgreSQL  
- Кластер PostgresSQL
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Load%20testing%20PostgreSQL/5.1.png)

- Инициализирую pgbench
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Load%20testing%20PostgreSQL/5.2.png)  

- Настройка кластера PostgreSQL
Применяю настройки, которые подобрал при помощи https://pgtune
```
-- DB Version: 17
-- OS Type: linux
-- DB Type: web
-- Total Memory (RAM): 15 GB
-- CPUs num: 6
-- Connections num: 100
-- Data Storage: ssd

ALTER SYSTEM SET
 max_connections = '100';
ALTER SYSTEM SET
 shared_buffers = '3840MB';
ALTER SYSTEM SET
 effective_cache_size = '11520MB';
ALTER SYSTEM SET
 maintenance_work_mem = '960MB';
ALTER SYSTEM SET
 checkpoint_completion_target = '0.9';
ALTER SYSTEM SET
 wal_buffers = '16MB';
ALTER SYSTEM SET
 default_statistics_target = '100';
ALTER SYSTEM SET
 random_page_cost = '1.1';
ALTER SYSTEM SET
 effective_io_concurrency = '200';
ALTER SYSTEM SET
 work_mem = '37095kB';
ALTER SYSTEM SET
 huge_pages = 'off';
ALTER SYSTEM SET
 min_wal_size = '1GB';
ALTER SYSTEM SET
 max_wal_size = '4GB';
ALTER SYSTEM SET
 max_worker_processes = '6';
ALTER SYSTEM SET
 max_parallel_workers_per_gather = '3';
ALTER SYSTEM SET
 max_parallel_workers = '6';
ALTER SYSTEM SET
 max_parallel_maintenance_workers = '3';
```

- Делаю рестарт кластера и запускаю проверку
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Load%20testing%20PostgreSQL/5.3.png)

- Лучший результат который удалось получить
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Load%20testing%20PostgreSQL/5.6.png)

