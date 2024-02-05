# Логический уровень PostgreSQL. Работа с базами данных, пользователями и правами

## Подготовка

### Создание ВМ

1. Заходим в Yandex.Cloud
2. Выбираем Все сервисы > Compute Cloud > Создать ВМ
3. Создаём сеть:  
3.1. В поле Каталог указываем "default"  
3.2. В поле "Имя" "otus-vm-db-pg-net-1"  
3.3. Выбираем "Создать подсети"  
3.4. Нажимаем "Создать"  
4. Заполняем имя ВМ "otus-db-pg-vm-1"
5. В поле "Зона доступности" указываем "ru-central1-d"
6. В разделе "Операционные системы" выбираем "Ubuntu 20.04"
7. В поле "RAM" ставим "4 ГБ"
8. В поле "Логин" указываем **otus**  
9. В поле "SSH-ключ" указываем публичный ключ:  
9.1. Если не генерировали SSH-ключ ранее:  
9.1.1. Заходим в PuttyGen  
9.1.2. Нажимаем "Generate"  
9.1.3. Генерируем ключи, двигая мышкой по пустой области  
9.1.4. Сохраняем публичный и приватный ключи в отдельную папку  
9.1.5. Копируем текст из поля Key (Public key for pasting into Open SSH authorized_keys file)  
9.1.6. Вставляем в поле "SSH-ключ"  
9.2. Если уже есть сгенерированный ключ:  
9.2.1. Правый клик по файлу *.ppk > Edit with PuTTYGen  
9.2.2. Копируем текст из поля Key (Public key for pasting into Open SSH authorized_keys file)  
9.2.3. Вставляем в поле "SSH-ключ"  
10. Проставляем галочку "Разрешить доступ к серийной консоли"  
11. Нажимаем "Создать ВМ"  
12. В открывшемся окне ждём завершения создания ВМ и **сохраняем публичный IPv4 (158.160.130.93)**  

### Подключение к ВМ через Putty

1. Подгружаем двойным кликом *.ppk, чтобы использовать ключи для подключения
2. Открываем Putty
3. В поле Host Name указываем внешний IPv4, полученный при создании ВМ (в пункте 12)
4. Нажимаем Open
5. В появившемся окне выбираем Accept
6. На запрос логина вводим логин, указанный при создании ВМ (в пункте 8)  
  
Теперь можно выполнять команды.

### Установка и подключение к PostgreSQL:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```

Проверяем, что установка прошла успешно:
```
pg_lsclusters
```

В консоли выведется:
```
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

**Подключаемся к PostgreSQL**:
```
sudo -u postgres psql
```

## Задания

### 1. Создайте новый кластер PostgresSQL 14

Выполняем команду и ждём завершения выполнения:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14
```

### 2. Зайдите в созданный кластер под пользователем postgres

Выполняем команду
```
sudo -u postgres psql
```

В консоль выведется:
```
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
Type "help" for help.
```
Теперь можно вводить команды SQL.

### 3. Создайте новую базу данных testdb

```sql
create database testdb;
```

В консоль выведется:
```
CREATE DATABASE
```

### 4. Зайдите в созданную базу данных под пользователем postgres

postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".

### 5. Создайте новую схему testnm

testdb=# create schema testnm;
CREATE SCHEMA

### 6. Создайте новую таблицу t1 с одной колонкой c1 типа integer

create table t1 (c1 integer);
CREATE TABLE


### 7. Вставьте строку со значением c1=1

insert into t1 values(1);
INSERT 0 1


### 8. Создайте новую роль readonly

create role readonly;
CREATE ROLE


### 9. Дайте новой роли право на подключение к базе данных testdb

grant connect on DATABASE testdb TO readonly;
GRANT


### 10. Дайте новой роли право на использование схемы testnm

grant usage on SCHEMA testnm to readonly;
GRANT

### 11. Дайте новой роли право на select для всех таблиц схемы testnm

grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT


### 12. Создайте пользователя testread с паролем test123

create user testread with password 'test123';
CREATE ROLE


### 13. Дайте роль readonly пользователю testread

grant readonly TO testread;
GRANT ROLE

### 14. Зайдите под пользователем testread в базу данных testdb

testdb=# \q
otus@otus-vm-db-pg-net-1:~$ psql -h localhost -U testread -d testdb -W
Password: test123
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

### 15. Сделайте select * from t1;

select * from t1;
ERROR:  permission denied for table t1

### 16. Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

Таблица в схеме public, а права давали на схему testnm
надо было создавать в другой схеме

### 17. Напишите что именно произошло в тексте домашнего задания

### 18. У вас есть идеи почему? ведь права то дали?

### 19. Посмотрите на список таблиц

### 20. Подсказка в шпаргалке под пунктом 20

### 21. А почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)

### 22. Вернитесь в базу данных testdb под пользователем postgres

testdb=> \q
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres psql -d testdb
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
Type "help" for help.


### 23. Удалите таблицу t1

drop table t1;
DROP TABLE


### 24. Создайте ее заново но уже с явным указанием имени схемы testnm

create table testnm.t1(c1 integer);
CREATE TABLE

### 25. Вставьте строку со значением c1=1

insert into testnm.t1 values(1);
INSERT 0 1


### 26. Зайдите под пользователем testread в базу данных testdb

testdb=# \q
otus@otus-vm-db-pg-net-1:~$ psql -h localhost -U testread -d testdb -W
Password: test123
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.


### 27. Сделайте select * from testnm.t1;

select * from testnm.t1;
ERROR:  permission denied for table t1

### 28. Получилось?

Нет.

### 29. Есть идеи почему? если нет - смотрите шпаргалку

testnm.t1 - новая таблица, доступ к ней не выдавался.

### 30. Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

Чтобы такое не повторялось для новых таблиц, зададим дефолтную роль на SELECT
testdb=> \q
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres psql -d testdb
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
Type "help" for help.

testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES

### 31. Сделайте select * from testnm.t1;

\q
psql -h localhost -U testread -d testdb -W
Password: test123
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

select * from testnm.t1;
ERROR: permission denied for table t1

### 32. Получилось?

Нет.

### 33. Есть идеи почему? если нет - смотрите шпаргалку

Определение дефолтной роль на SELECT решает проблему для новых таблиц, а таблица testnm.t1 была создана ранее. Дадим роль всем текущим таблицам в схеме
testdb=> \q
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres psql -d testdb
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
Type "help" for help.

testdb=# grant SELECT on all TABLES in SCHEMA testnm TO readonly;
GRANT

### 34. Сделайте select * from testnm.t1;

\q
psql -h localhost -U testread -d testdb -W
Password: test123
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

select * from testnm.t1;
 c1
----
  1
(1 row)

### 35. Получилось?

Да.

### 36. Ура!

Ура!

### 37. Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1

### 38. А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

Выполнив команду \dt можем заметить, что таблица создана в схеме public. Пользователь обладает правами на неё.

testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t2   | table | testread
(1 row)

### 39. Есть идеи как убрать эти права? если нет - смотрите шпаргалку

Заходим в psql под ролью postgres:

testdb=> \q
otus@otus-vm-db-pg-net-1:~$ sudo -u postgres psql -d testdb
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
Type "help" for help.

Удаляем дефолтные права роли public на схему public:
testdb=# REVOKE CREATE on SCHEMA public FROM public;
REVOKE

Удаляем дефолтные права роли public на базу testdb:
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE

### 40. Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему, выполнив указанные в ней команды


### 41. Теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

\q
psql -h localhost -U testread -d testdb -W
Password: test123
psql (14.10 (Ubuntu 14.10-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
INSERT 0 1

### 42. Расскажите что получилось и почему

В консоль вывелась ошибка об отсутствии прав, как и ожидалось.
