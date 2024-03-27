# MVCC, vacuum и autovacuum.  Настройка autovacuum с учетом особеностей производительности
### Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB

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
8. Нажать кнопку "Готово"  
9. Подождать, пока установится ОС  
10. Пропустить все вопросы, связанные с настройкой, после запуска ОС  
11. Терминал при этом не открывается, поэтому нужно зайти в Settings > Region > Language > Login Screen > явным образом выбрать English (United States) без ISO-8859-1 и перезагрузить ОС (см. https://ru.stackoverflow.com/questions/1476869/%D0%9D%D0%B5-%D0%BC%D0%BE%D0%B3%D1%83-%D0%BE%D1%82%D0%BA%D1%80%D1%8B%D1%82%D1%8C-%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B0%D0%BB-%D0%B2-ubuntu)  
12. Админских прав у пользователя не будет, поэтому нужно (см. https://youtu.be/jZGHtuxpaP4?si=X6mA74Qdt00wUaw6):  
12.1. Ввести команду su  
12.2. Ввести пароль текущего пользователя  
12.3. Ввести команду usermod -aG sudo vboxuser  
12.4. Перезагрузить ОС, например, командой reboot  
13. Даже если настроить в VirtualBox двунаправленный буфер обмена, он работать не будет. Чтобы это исправить, нужно:  
13.1. Скачать образ https://download.virtualbox.org/virtualbox/6.1.2/VBoxGuestAdditions_6.1.2.iso  
13.2. Смонтировать образ, например, открыв его в папке, с помощью двойного клика  
13.3. Открыть терминал в папке смонтированного образа  
13.4. Выполнить команду sudo sh VBoxLinuxAdditions.run  
13.5. Перезагрузить ОС  
13.6. После этого на экране будут постоянно появляться странные артефакты, но по крайней мере заработает общий буфер обмена. После дополнительной пары перезагрузок артефакты могут пропасть.  


### Установить на него PostgreSQL 15 с дефолтными настройками

В этом поможет следующий набор команд:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Появится длинный лог установки, можем его пропустить.

### Создать БД для тестов

Для этого нужно выполнить команду:  
```
sudo -u postgres pgbench -i postgres
```

В консоль выведется:  
```
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.06 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.22 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.08 s, vacuum 0.03 s, primary keys 0.08 s).
```

### Запустить pgbench
```
sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres
```

В консоль выведется:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 782.1 tps, lat 10.116 ms stddev 7.050, 0 failed
progress: 12.0 s, 785.9 tps, lat 10.121 ms stddev 6.400, 0 failed
progress: 18.0 s, 787.2 tps, lat 10.109 ms stddev 6.477, 0 failed
progress: 24.0 s, 789.8 tps, lat 10.063 ms stddev 6.408, 0 failed
progress: 30.0 s, 766.5 tps, lat 10.394 ms stddev 6.332, 0 failed
progress: 36.0 s, 742.6 tps, lat 10.724 ms stddev 5.965, 0 failed
progress: 42.0 s, 755.3 tps, lat 10.537 ms stddev 6.037, 0 failed
progress: 48.0 s, 749.0 tps, lat 10.614 ms stddev 6.786, 0 failed
progress: 54.0 s, 758.5 tps, lat 10.501 ms stddev 6.230, 0 failed
progress: 60.0 s, 782.8 tps, lat 10.161 ms stddev 6.516, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 46207
number of failed transactions: 0 (0.000%)
latency average = 10.330 ms
latency stddev = 6.436 ms
initial connection time = 14.382 ms
tps = 769.981779 (without initial connection time)
```

### Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
В этом поможет редактор nano:
```
sudo nano /etc/postgresql/15/main/postgresql.conf
```

В консоль выведется весь файл, его можно скорллить и редактировать.  
Для поиска настройки можно нажать Ctrl+W, ввести её название, нажать enter и отредактировать найденную строку.  
Задаём настройки, как указано в прикреплённом файле к ДЗ.  
Получившийся файл с группировокой по разделам представлен ниже (за исключением параметров, оставшихся без изменения):  
```
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

max_connections = 40

#------------------------------------------------------------------------------
# RESOURCE USAGE (except WAL)
#------------------------------------------------------------------------------

shared_buffers = 1GB
work_mem = 6553kB
maintenance_work_mem = 512MB
effective_io_concurrency = 2

#------------------------------------------------------------------------------
# QUERY TUNING
#------------------------------------------------------------------------------

random_page_cost = 4
effective_cache_size = 3GB
default_statistics_target = 500

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

wal_buffers = 16MB
checkpoint_completion_target = 0.9
min_wal_size = 4GB
max_wal_size = 16GB
```

### Протестировать заново
Запускаем команду:
```
sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres
```

В консоли видим:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 805.6 tps, lat 9.829 ms stddev 6.304, 0 failed
progress: 12.0 s, 800.0 tps, lat 9.948 ms stddev 5.974, 0 failed
progress: 18.0 s, 800.4 tps, lat 9.934 ms stddev 6.067, 0 failed
progress: 24.0 s, 805.4 tps, lat 9.874 ms stddev 6.327, 0 failed
progress: 30.0 s, 792.9 tps, lat 10.040 ms stddev 6.129, 0 failed
progress: 36.0 s, 811.3 tps, lat 9.791 ms stddev 6.428, 0 failed
progress: 42.0 s, 789.8 tps, lat 10.088 ms stddev 6.112, 0 failed
progress: 48.0 s, 783.0 tps, lat 10.172 ms stddev 5.920, 0 failed
progress: 54.0 s, 787.4 tps, lat 10.088 ms stddev 6.511, 0 failed
progress: 60.0 s, 789.1 tps, lat 10.084 ms stddev 6.139, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 47798
number of failed transactions: 0 (0.000%)
latency average = 9.985 ms
latency stddev = 6.198 ms
initial connection time = 13.003 ms
tps = 796.558377 (without initial connection time)
```

### Что изменилось и почему?
Обработалось на 1500 транзакций больше за 1 минуту. Разница не слишком большая, может быть связана как с погрешностью измерения, так и с тем, что по сравнению со стандартными настройками по большей части памяти стало больше, а уменьшение некоторых параметров не сыграло критичной роли.

### Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
Запускаем psql:
```
sudo -u postgres psql
```
В консоль выведется:
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```
Теперь можем вводить команды SQL:  
```sql
create table mytable (data varchar(100));
insert into mytable (select * from generate_series(1, 1000000) as value);
```

В консоль выведется:  
```
CREATE TABLE
INSERT 0 1000000
```

### Посмотреть размер файла с таблицей

В этом поможет команда:
```sql
select pg_size_pretty(pg_TABLE_size('mytable'));
```

В консоли видим размер таблицы 35 Мб:  
```
 pg_size_pretty 
----------------
 35 MB
(1 row)
```

### 5 раз обновить все строчки и добавить к каждой строчке любой символ
Для этого можно 5 раз выполнить команду:
```sql
update mytable set data = data || '0';
```
В консоль каждый раз после некоторой паузы будет выводиться:
```
UPDATE 1000000
```
### Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
В этом поможет команда:
```sql
select relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum from pg_stat_user_tables where relname = 'mytable';
```
В консоль выведется:
```
 relname | n_live_tup | n_dead_tup | ratio% | last_autovacuum 
---------+------------+------------+--------+-----------------
 mytable |    1000000 |    4999870 |    499 | 
(1 row)
```

### Подождать некоторое время, проверяя, пришел ли автовакуум

```
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 mytable |    1001956 |          0 |      0 | 2024-03-28 00:12:13.272662+03
(1 row)
```

### 5 раз обновить все строчки и добавить к каждой строчке любой символ
Снова 5 раз выполним команду:
```sql
update mytable set data = data || '0';
```
В консоль каждый раз после некоторой паузы будет выводиться:
```
UPDATE 1000000
```
### Посмотреть размер файла с таблицей
Запускаем:
```sql
select pg_size_pretty(pg_TABLE_size('mytable'));
```

Видим размер таблицы: 
```
 pg_size_pretty 
----------------
 260 MB
(1 row)
```
### Отключить Автовакуум на конкретной таблице
В этом поможет команда:  
```sql
alter table mytable set (autovacuum_enabled = false);
```

В консоль выведется:  
```
ALTER TABLE
```
### 10 раз обновить все строчки и добавить к каждой строчке любой символ
 Теперь 10 раз выполним команду:
```sql
update mytable set data = data || '0';
```
В консоль каждый раз после некоторой паузы будет выводиться:
```
UPDATE 1000000
```
### Посмотреть размер файла с таблицей
Запускаем:  
```sql
select pg_size_pretty(pg_TABLE_size('mytable'));
```

Видим размер таблицы:  
```
 pg_size_pretty 
----------------
 569 MB
(1 row)
```
### Объясните полученный результат

При обновлении записей каждый раз создаётся новая строка, а старая помечается флагом и ждёт очистки, например, через vacuum или autovacuum.  
Очистка физически не удаляет данные с диска, но позволяет переиспользовать освобождённое пространство.  
Так как autovacuum отключен, старые строки остаются и значительно увеличивают размер таблицы.  
Физически размер таблицы уменьшает только только vacuum full, поэтому размер таблицы не уменьшается, пока vacuum full не будет вызван.  
При вызове vacuum full будет создан новый файл таблицы, в которую будут перенесены все актуальные данные, а старый будет удалён.  

### Не забудьте включить автовакуум

Выполняем уже знакомую команду:
```sql
alter table mytable set (autovacuum_enabled = true);
```

В консоль выведется:
```
ALTER TABLE
```
### Задание со *: Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. Не забыть вывести номер шага цикла.

Выполним команду:
```sql
do $$declare index int;
begin
    for index in 1..10
    loop
        raise info 'Step %', index;
        update mytable set data = data || '0';
    end loop;
end$$;
```
Таблица сразу начнёт меняться, а в консоль будут выводиться итерации:
```
INFO:  Step 1
INFO:  Step 2
INFO:  Step 3
INFO:  Step 4
INFO:  Step 5
INFO:  Step 6
INFO:  Step 7
INFO:  Step 8
INFO:  Step 9
INFO:  Step 10
DO
```
