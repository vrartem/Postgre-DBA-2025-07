- Демонстрационную базу данных взял тут: https://postgrespro.ru/education/demodb

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.2.png)

- Прямое соединение двух или более таблиц. Список рейсов с деталями маршрута и данными об аэропорте вылета

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.2.png)

- Левостороннее соединение двух или более таблиц:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/9.4.png)

  В данном примере искусственно создал условие, при котором две первых записи имеют значение null.

- Кросс соединение двух или более таблиц:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.5.png)

	В этом примере количество строк = (количество рейсов) × (количество самолётов).

- Полное соединение двух или более таблиц:

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.3.png)

	В данном случае FULL JOIN будет равнозначен INNER JOIN, для условия f.route_no = r.route_no всегда есть одна или более записий.

- Разные типы соединений:

  Пробую сравнить цены относительно средней цены за месяц

![](https://raw.githubusercontent.com/vrartem/Postgre-DBA-2025-07/refs/heads/main/Autovacuum/6.6.png)

