- Для подлючения использую DBeaver и PGAdmin

- Проверяю текущий уровень изоляции:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Isolation%20levels/Screenshot%202025-08-15%20132134.png)

- Между двумя подключениями проверяем транзакции на текущем уровне изоляции (Read Сommitted).

  Выполняю последовательность из инструкции:
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Isolation%20levels/READ_COMMITTED.png)

  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Isolation%20levels/READ_COMMITTED_1.1.png)

- В Read Сommitted первая транзакция не может читать еще не зафиксированные изменения в другой транзакции, после (COMMIT) изменения будут видны для первой транзакции (Да для Неповторяющееся чтение). 

- Пробуем для repeatable read:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Isolation%20levels/Screenshot%202025-08-15%20140719.png)

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Isolation%20levels/Screenshot%202025-08-15%20160225.png)

- Для Repeatable Read будет тоже самое, только изменения будут доступны после (COMMIT) первой и второй транзакции. (Нет для Неповторяющееся чтение)



