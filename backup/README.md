- Создал БД, схему и таблицу.
- Заполнил таблицу автосгенерированными 100 записями.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/backup/12.1.png)

- Под линукс пользователем Postgres создал каталог для бэкапов

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/backup/12.2.png)

- Сделал логический бэкап используя утилиту COPY

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/backup/12.3.png)

- Восстановил в 2 таблицу данные из бэкапа.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/backup/12.4.png)

- Используя утилиту pg_dump создал бэкап в сжатом формате двух таблиц

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/backup/12.6.png)

- Используя утилиту pg_restore восстановил в новую БД только вторую таблицу!

	При выполнении команды `pg_restore -d test2 -n testschema -t table2 -U postgres -p 5432 /tmp/postgres_backups/backup2.gz` возникала ошибка, не мог создать схему.
	После создания схемы вручную все получилось.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/backup/12.5.png)
