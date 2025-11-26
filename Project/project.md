## 1. Конфигурация виртуальной машины


- **CPU**: 4 ядра  
- **Оперативная память**: 6ГБ   
- **Диск**: 120ГБ SSD


## 2. Установка кластера PostgreSQL

### Шаг 1. Установка пакетов


Выполните команду для установки PostgreSQL и дополнительных модулей:

```
sudo apt install postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql
```