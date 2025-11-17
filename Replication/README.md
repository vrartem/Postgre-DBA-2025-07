- В Docker Desktop создаю три контейнера postgres, порты 5432, 5433, 5434,
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Replication/14.3.png)

- Настройки для всех кластеров

1. В postgresql.conf: wal_level = logical
2. В pg_hba.conf:	host    reolication     all             172.17.0.0/0            md5 
	
- Таблицы для всех кластеров
  ```
  CREATE TABLE test (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW());
  
  CREATE TABLE test2 (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW());
	```

- Создание PUBLICATION и SUBSCRIPTION выполнил по условию ДЗ, исопльзовал следующие команды
  ```
   CREATE PUBLICATION pub_xxx FOR TABLE testx;

   CREATE SUBSCRIPTION sub_test_xxx
   CONNECTION 'host=172.17.0.x port=543x user=postgres password=postgres dbname=postgres'
   PUBLICATION pub_xxx;
  ```
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Replication/14.2.png)

- Использовал DBeaver, создал три подключения, пробовал вставлять данные и проверять данные в других таблицах по связям PUBLICATION - SUBSCRIPTION

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Replication/14.4.png)

Например: "Тест из pg1" "2025-11-14 20:18:21.544" - created_at будет одинаковая, хотя она TIMESTAMP DEFAULT NOW()

- Столкнулся со следующими проблемами:
  
1. CONNECTION host=localhost, localhost не работает, для docker контейнеров нужно смотреть сеть, в моем случае 172.17.0.0
2. Вторая проблема с которой я провозился, не мог установить соединение моежду pg2 b pg3, оказывается у двух контейнеров одинаковый mac адрес был.
  После редактирование mac для контейнера все заработало
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Replication/14.5.png)
