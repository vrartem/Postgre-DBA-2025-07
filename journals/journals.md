-  Ставлю контрольные точки раз в 30 секунд:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.1.png)

- Запускаю PGBENCH: 

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.2.png)

- Анализ журналов:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.4.png)
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.5.png)
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.6.png)

Вижу, что через каждые 30с запускался checkpoint, за 10мин должно было выпонится 20 чекпоинтов.
Каждый Wal файл размером по 16мб. Общий объем 320мб.

- Запускою PGBENCH в асинхронном режиме synchronous_commit=off

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.7.png)

- Заново создаю кластер с включенной котрольной суммой, cоздаю таблицу и вставляю данные

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.8.png)

- Правлю таблицу и пробую:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/journals/7.9.png)

После модификации таблицы crc сумма не проходит проверку.
Проигнорировать ошибку можно установив заначение "on" для ignore_checksum_failure
