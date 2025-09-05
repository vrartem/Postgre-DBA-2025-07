## Логический уровень PostgreSQL
- Проверяю кластер PostgresSQL
 ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Logical%20Layer/4.1.png)

- Выполняю пункты (2-13)
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Logical%20Layer/4.2.png)

- Захожу под пользователем в БД и выполняю селект, получаю ошибку доступа
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Logical%20Layer/4.3.png)
  
  таблица создается по умолчанию в схеме public, но прав на схему public новому пользователю не давались.

- Пересоздаю таблицу с данными, пытаюсь прочитать данные:
  Снова получаю ошибку, так как ранее выданные гранты не распостраняются на новую таблицу.
  Выполняю grant SELECT on ALL TABLES in SCHEMA testnm TO readonly;
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Logical%20Layer/4.4.png)
  SELECT выполнися успешно
  Чтобы эта проблема не повторялась с новыми таблицами, может создать привилегию по умолчанию:
  ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;

- Важный момент! Новым пользователям роль public добавляется автоматически, благодаря этому, она может создавать даже новые таблицы.
  Крайне важно отозвать у таких пользователей права через команду REVOKE
  Делаю:
  REVOKE CREATE on SCHEMA public FROM public; 
  REVOKE ALL on DATABASE testdb FROM public;
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Logical%20Layer/4.5.png)
  Теперь доспупа нет
