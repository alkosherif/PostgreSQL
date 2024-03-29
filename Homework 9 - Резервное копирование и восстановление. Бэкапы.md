# Настройка PostgreSQL. Нагрузочное тестирование и тюнинг PostgreSQL	
### Создаем ВМ/докер c ПГ.

#### Создание ВМ
Для выполнения ДЗ будем использовать **VirtualBox** и образ **Ubuntu 22.04 LTS**, они доступны бесплатно по первой ссылке в гугле.  
Для запуска ОС потребуется:  
1. Скачать образ Ubuntu 22.04.4 LTS отсюда: https://ubuntu.com/download/desktop  
2. Установить VirtualBox отсюда: https://www.virtualbox.org/wiki/Downloads  
3. Открыть VirtualBox  
4. Нажать в главном меню Машина > Создать  
5. Открыть раздел "Имя и тип ОС"  
5.1. В выпадающем списке "Образ ISO" выбрать "Другой" и указать скачанный ранее iso-образ  
5.2. В выпадающем списке "Тип" выбрать "Linux"  
5.3. В выпадающем списке "Версия" выбрать "Ubuntu 22.04 LTS (Jammy Jellyfish) (64-bit)"  
5.4. В поле "Имя" ввести любое имя  
6. Открыть раздел "Автоматическая установка"  
6.1. В поле "Имя хоста" указать любое корректное имя без пробелов  
6.2. В полях "Пароль" и "Подтвердите пароль" задать пароль, либо нажать на глаз и запомнить пароль, который там был  
7. Открыть раздел "Оборудование"  
7.1 Выбрать Основную память 8192 МБ  
7.2 Выбрать 2 процессора  
8. Открыть раздел "Жёсткий диск"  
8.1 В разделе "Расположение и размер файла жёсткого диска" выбрать 25 ГБ  
9. Нажать кнопку "Готово"  
10. Подождать, пока установится ОС  
11. Пропустить все вопросы, связанные с настройкой, после запуска ОС  
12. Терминал при этом не открывается, поэтому нужно зайти в Settings > Region > Language > Login Screen > явным образом выбрать English (United States) без ISO-8859-1 и перезагрузить ОС (см. https://ru.stackoverflow.com/questions/1476869/%D0%9D%D0%B5-%D0%BC%D0%BE%D0%B3%D1%83-%D0%BE%D1%82%D0%BA%D1%80%D1%8B%D1%82%D1%8C-%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B0%D0%BB-%D0%B2-ubuntu)  
13. Админских прав у пользователя не будет, поэтому нужно (см. https://youtu.be/jZGHtuxpaP4?si=X6mA74Qdt00wUaw6):  
13.1. Ввести команду su  
13.2. Ввести пароль текущего пользователя  
13.3. Ввести команду usermod -aG sudo vboxuser  
13.4. Перезагрузить ОС, например, командой reboot  
14. Даже если настроить в VirtualBox двунаправленный буфер обмена, он работать не будет. Чтобы это исправить, нужно:  
14.1. Скачать образ https://download.virtualbox.org/virtualbox/6.1.2/VBoxGuestAdditions_6.1.2.iso  
14.2. Смонтировать образ, например, открыв его в папке, с помощью двойного клика  
14.3. Открыть терминал в папке смонтированного образа  
14.4. Выполнить команду sudo sh VBoxLinuxAdditions.run  
14.5. Перезагрузить ОС  
14.6. После этого на экране ВМ будут постоянно появляться странные артефакты, но по крайней мере заработает общий буфер обмена. При изменении размеров окна ВМ проблема пропадает.

#### Установка PostgreSQL 15

В этом поможет следующий набор команд:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Появится длинный лог установки, можем его пропустить.

### Создаем БД, схему и в ней таблицу.
Заходим в psql:
```
sudo -u postgres psql
```
Теперь можем вводить SQL-команды.  
Создаём БД и схему:  
```sql
create database my_database;
```
В консоль выведется:
```
CREATE DATABASE
```
Переходим в созданную БД:
```sql
\c my_database
```
В консоль выведется:
```
You are now connected to database "my_database" as user "postgres".
```
Создаём схему:
```sql
create schema my_schema;
```
В консоль выведется:
```
CREATE SCHEMA
```
Создаём таблицу:
```sql
create table my_schema.my_table (data int);
```
В консоль выведется:
```
CREATE TABLE
```
### Заполним таблицы автосгенерированными 100 записями.
```sql
insert into my_schema.my_table select random() * 1000 from generate_series(1,100);
```
В консоль выведется:
```
INSERT 0 100
```
Посмотрим, какие данные сейчас в таблице для сравнения с будущим бэкапом:
```sql
select * from my_schema.my_table limit 10;
```
В консоль выведется:
```
  data 
------
  226
  862
  895
  641
  738
  775
  777
  571
  461
  682
(10 rows)
```
### Под линукс пользователем Postgres создадим каталог для бэкапов

Выйдем из psql:
```sql
\q
```
Создадим каталог для бэкапов:
```
sudo mkdir /mnt/backups
```

Сделаем пользователя postgres его владельцем:
```
sudo chown postgres /mnt/backups
```

### Сделаем логический бэкап используя утилиту COPY
Снова заходим в psql:  
```
sudo -u postgres psql
```
Перейдём в созданную БД:  
```sql
\c my_database
```
В консоль выведется:  
```
You are now connected to database "my_database" as user "postgres".
```
Сделаем бэкап:  
```sql
\copy my_schema.my_table to '/mnt/backups/my_table_backup1.sql'
```
В консоль выведется:
```
COPY 100
```
### Восстановим в 2 таблицу данные из бэкапа.
Создадим 2 таблицу:
```sql
create table my_schema.my_table_from_backup (data int);
```
В консоль выведется:
```
CREATE TABLE
```
Теперь восстановим в неё данные из бэкапа:
```sql
\copy my_schema.my_table_from_backup from '/mnt/backups/my_table_backup1.sql'
```
В консоль выведется:
```
COPY 100
```
Посмотрим, соответствуют ли данные тем, что были раньше:
```sql
select * from my_schema.my_table_from_backup limit 10;
```
В консоли увидим, что данные в таблицы те же самые:
```
  data 
------
  226
  862
  895
  641
  738
  775
  777
  571
  461
  682
(10 rows)
```
### Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
### Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
