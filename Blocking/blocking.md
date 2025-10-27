- Меняю настройки log_lock_waits и deadlock_timeout

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Blocking/8.png)

- Создаю новую талбицу, вставляю несколько записей и симулирую блокировку.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Blocking/8.1.png)

- Проверяю логи, вижу записи блокировок:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Blocking/8.2.png)

- Выполнил update в трех сессиях и вывел информацию из pg_locks в удобном виде:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Blocking/8.3.png)

Видно, что процесс 10726 заблокировал таблицу в RowExclusiveLock, процесс 10672 ожидает 10726, а процесс 10721 ожидает 10672
RowExclusiveLock блокировка на уровне таблицы
ExclusiveLock блокировка на уровне строки

- Делаю взаимоблокировку трех транзакций и вижу ошибку в psql

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Blocking/8.4.png)

Если проанализировать лог, то можно увидеть, что происходило с таблицей, какие запросы выполнялись, какие пороцессы ожидают друг друга.

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Blocking/8.5.png)

- Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Cначала будет заблокирована таблица для первой транзакции, после commit, будет выполнен update из другой транзакции. 



