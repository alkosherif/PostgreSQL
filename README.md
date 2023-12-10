# Подготовка

### Создание ВМ

Т.к. чтобы использовать линуксовые команды в командной строке Windows, понадобится WSL, а его надо устанавливать перед созданием ВМ (т.к. потребуется перезагрузка), воспользуемся Putty:
1. Выбираем в Яндекс-Облаке Compute cloud -> Создать ВМ1
2. Добавляем сеть
3. Расставляем рекомендованные параметры из видео по занятию (Ubuntu 20, 15 Гб HDD, 4 Гб RAM)
4. Генерируем ключ ssh:
  * Устанавливаем Putty
  * Заходим в PuttyGen
  * Генерируем ключи, двигая мышкой по пустой области
  * Сохраняем публичный и приватный ключи в отдельную папку
5. Публичный ключ из сохранённого файла указываем в настройках ВМ
6. Нажимаем кнопку создать

### Подключение к PostgreSQL в ВМ

1. Открываем сохранённый приватный .ppk в проводнике мышкой
2. Копируем внешний IPv4 адрес из информации о ВМ
3. Вбиваем его в PUTTY
4. Устанавливаем Postgre 16
  > install postgresql-16
5. Проверяем, что всё установлено, командой pg_lsclusters (видим 1 кластер)
6. Запускаем psql командой:
  > sudo -u postgres psql

# Выполнение скриптов в PostgreSQL

Отключаем автокоммит в обеих сессиях командой:
> \set AUTOCOMMIT OFF

В консоль ничего не выведется.
Проверяем, что автокоммит отключен:
> \echo :AUTOCOMMIT

В консоль выведется:
> OFF

Создаём БД:
```SQL
CREATE DATABASE iso;
```

В консоли появится:
>CREATE DATABASE

Выбираем созданную БД:
>\c iso

В консоли появится:
>You are now connected to database "iso" as user "postgres".

Проверяем, что подключились к нужной базе:
```SQL
SELECT current_database();
```
 
В консоли выведется:
```
current_database
------------------
iso
(1 row)
```

В первой сессии выполняем команды:
```SQL
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```

В консоль выведется:
```SQL
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```

Выполняем команду:
```SQL
show transaction isolation level
```

Получаем ответ:
```
 transaction_isolation
-----------------------
 read committed
(1 row)
```

Начинаем новую транзакцию командой:
```SQL
begin transaction;
```
В консоль выведется:
>BEGIN

В сессии 1 добавляем новую запись:
```SQL
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```

Во сессии 2 выполняем команду:
```SQL
select * from persons;
```

Получаем:
 ```
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

**Новая запись отсутствует, так как транзакция в первой сессии ещё не закоммичена, а текущий уровень изоляции транзакций - чтение только закоммиченных транзакций.**

Выполняем коммит транзакции в сессии 1:
```SQL
commit;
```
В консоль выведется:
>COMMIT

Во сессии 2 выполняем команду:
```SQL
select * from persons;
```
Получаем:
 ```
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

**Новая запись появилась во сессии 2, поскольку транзакция в сессии 1 была закоммичена и теперь эти данные доступны всем.**

Завершаем транзакцию в сессии 2 командой:
```SQL
commit transaction;
```

В консоль выведется:
>COMMIT

В **обеих** сессиях устанавливаем уровень изоляции транзакций repeatable read:
```SQL
set transaction isolation level repeatable read;
```

В консоль выведется:
>SET

В сессии 1 добавляем новую запись:
```SQL
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
В консоль выведется:
>INSERT 0 1

В сессии 2 смотрим таблицу:
```SQL
select * from persons;
```

В консоль выводится:
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Новой записи нет, потому что **?????????**

Выполняем коммит транзакции в сессии 1:
```SQL
commit;
```

В консоль выведется:
>COMMIT

В сессии 2 смотрим таблицу:
```SQL
select * from persons;
```

В консоль выводится:
 ```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Новой записи нет, потому что **?????????**

Выполняем коммит транзакции в сессии 2:
```SQL
commit;
```

В консоль выведется:
>COMMIT

Запрашиваем таблицу в сессии 2 командой:
```SQL
select * from persons;
```

Видим результат:
```SQL
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Новая запись появилась, потому что **?????????????????**
