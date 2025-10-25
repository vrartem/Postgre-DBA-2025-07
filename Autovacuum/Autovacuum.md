- Инициализация pgbench

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.2.png)

- Применил настройки и запустил pgbench:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.3.png)

- Сделал RESET и запустил повторно:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.4.png)

При применении предоставленных настроек количество tps получаю меньше. 
Это может быть связано с тем, что avtovacuum стал давать допонительную нагрузку.
На это могли повлиять настройки maintenance_work_mem, min\max_wal_size, wal_buffers

- Создал и заполнил таблицу, вывел размер:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.5.png)

- Количество метрвых строк после update:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.6.png)

- Проверяю повторно:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.7.png)

- Еще раз делаю update выключаю autovacuum и снова update, видим, что размер увеличился:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.8.png)

