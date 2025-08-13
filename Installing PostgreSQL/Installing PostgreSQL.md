- Установил WSL и Docker Desktop
- Проверяю наличие установленного докера:
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20160509.png)

- Создаю каталог /var/lib/postgres (скрин сдела позже):
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-12%20163519.png)
- Развертываю контейнер с PostgreSQL 17 смонтировав в /var/lib/postgresql и пробую сразу подлючиться:
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20161158.png)

- Развернул контейнер с клиентом postgres, подключился из контейнера с клиентом к контейнеру с сервером, выполнил селект ранее вставленной записи в test:
  ![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20162704.png)

- Подключился к контейнеру через DBeaver:
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20163352.png)

- Удалил контейнер с сервером и создал заново, все данные на месте (запись test2 была добавлена ранее):
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20163918.png)
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20165016.png)
![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Installing%20PostgreSQL/Screenshot%202025-08-13%20165148.png)
