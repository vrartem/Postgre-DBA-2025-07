- Проверяю, что кластер запущен и создаю произвольную таблицу с произвольным содержимым

![]([https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Isolation%20levels/Screenshot%202025-08-15%20132134.png](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Postgres%20physical%20layer/3.2.png))

- Остановливаю postgres и создаю новый диск на 10gb

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Postgres%20physical%20layer/3.3.png)

- Переношу содержимое и пытаюсь запустить инстанс

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Postgres%20physical%20layer/3.4.png)

 - Получаю ошибку так как в /var/lib/postgres/17/main ничего нет
   
 - Правлю postgresql.conf и пробую запустить повторно и проверить данные

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Postgres%20physical%20layer/3.5.png)

- Данные на месте 
