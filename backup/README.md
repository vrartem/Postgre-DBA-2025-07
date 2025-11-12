- Создаем БД, схему и в ней таблицу.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)

- Заполним таблицы автосгенерированными 100 записями.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)

- Под линукс пользователем Postgres создадим каталог для бэкапов

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)

- Сделаем логический бэкап используя утилиту COPY

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)

- Восстановим в 2 таблицу данные из бэкапа.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)

- Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)

- Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!

	При выполнении команды `pg_restore -d test2 -n testschema -t table2 -U postgres -p 5432 /tmp/postgres_backups/backup2.gz` возникала ошибка, не мог создать схему.
	После создания схемы вручную все получилось.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/data_selection/9.2.png)