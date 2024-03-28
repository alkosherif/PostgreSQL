# Журналы. Работа с журналами
### Создание ВМ

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
14.6. После этого на экране ВМ будут постоянно появляться странные артефакты, но по крайней мере заработает общий буфер обмена.

### Установка PostgreSQL 15

В этом поможет следующий набор команд:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Появится длинный лог установки, можем его пропустить.

### Настройте выполнение контрольной точки раз в 30 секунд.

Выполним команду:  
```
sudo -u postgres psql
```

В консоль выведется:  
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```

Теперь можно выполнять команды SQL. Зададим интервал контрольной точки в 30 секунд и выйдем из psql:  
```sql
alter system set checkpoint_timeout = '30s';
\q
```

В консоль выведется:  
```
ALTER SYSTEM
```

Перезапустим кластер, чтобы изменения применились:
```
sudo pg_ctlcluster 15 main restart
```

### 10 минут c помощью утилиты pgbench подавайте нагрузку.

Наполним БД тестовыми данными:  
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
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.19 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.08 s, vacuum 0.03 s, primary keys 0.08 s).
```

Посмотрим параметры до запуска утилиты. Для этого зайдём в psql:
```
sudo -u postgres psql
```
И выполним команду:
```sql
select pg_current_wal_lsn(), pg_current_wal_insert_lsn(), pg_walfile_name(pg_current_wal_lsn()) as file_current_wal_lsn, pg_walfile_name(pg_current_wal_insert_lsn()) as file_current_wal_insert_lsn;
\q
```
В консоль выведется:
```
 pg_current_wal_lsn | pg_current_wal_insert_lsn |   file_current_wal_lsn   | file_current_wal_insert_lsn 
--------------------+---------------------------+--------------------------+-----------------------------
 0/21CDE98          | 0/21CDE98                 | 000000010000000000000002 | 000000010000000000000002
(1 row)
```

Теперь подадим нагрузку (8 клиентов, вывод прогресса каждые 60 секунд, время нагрузки 600 секунд):
```
sudo -u postgres pgbench -c8 -P 60 -T 600 postgres
```
В консоль выведется:

```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 803.1 tps, lat 9.870 ms stddev 7.562, 0 failed
progress: 120.0 s, 826.6 tps, lat 9.589 ms stddev 7.351, 0 failed
progress: 180.0 s, 831.1 tps, lat 9.537 ms stddev 7.407, 0 failed
progress: 240.0 s, 825.4 tps, lat 9.606 ms stddev 7.537, 0 failed
progress: 300.0 s, 791.4 tps, lat 10.031 ms stddev 7.330, 0 failed
progress: 360.0 s, 795.0 tps, lat 9.983 ms stddev 7.442, 0 failed
progress: 420.0 s, 811.8 tps, lat 9.769 ms stddev 7.203, 0 failed
progress: 480.0 s, 796.8 tps, lat 9.962 ms stddev 7.466, 0 failed
progress: 540.0 s, 805.9 tps, lat 9.844 ms stddev 7.528, 0 failed
progress: 600.0 s, 786.9 tps, lat 10.098 ms stddev 7.491, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 484449
number of failed transactions: 0 (0.000%)
latency average = 9.826 ms
latency stddev = 7.435 ms
initial connection time = 18.155 ms
tps = 807.408241 (without initial connection time)
```

Посмотрим параметры после работы утилиты. Для этого зайдём в psql:
```
sudo -u postgres psql
```
И выполним команду:
```sql
select pg_current_wal_lsn(), pg_current_wal_insert_lsn(), pg_walfile_name(pg_current_wal_lsn()) as file_current_wal_lsn, pg_walfile_name(pg_current_wal_insert_lsn()) as file_current_wal_insert_lsn;
```

В консоль выведется:
```
 pg_current_wal_lsn | pg_current_wal_insert_lsn |   file_current_wal_lsn   | file_current_wal_insert_lsn 
--------------------+---------------------------+--------------------------+-----------------------------
 0/2262D2C8         | 0/2262D2C8                | 000000010000000000000022 | 000000010000000000000022
(1 row)
```

### Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
Для измерения прироста объёма журнальных файлов понадобятся исходная и текущая позиции записи в журнале. Получаем разницу с помощью команды:
```
select pg_size_pretty('0/2262D2C8'::pg_lsn - '0/21CDE98'::pg_lsn) wal_size;
```
В консоль выведется:
```
 wal_size 
----------
 516 MB
(1 row)
```

Так как мы настроили выполнение контрольной точки раз в 30 секунд, а общее время теста - 600 секунд, следовательно, должно было создаться 600 / 30 = 20 контрольных точек.  
516 МБ / 20 = 25,8 МБ приходится на одну контрольную точку.  

### Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
В этом поможет команда:
```sql
select * from pg_stat_bgwriter; 
```

В консоль выведется:
```
 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint | buffers_clean | maxwritten_clean | buffers_backend | buffers_backend_fsync | buffers_alloc |          stats_reset          
-------------------+-----------------+-----------------------+----------------------+--------------------+---------------+------------------+-----------------+-----------------------+---------------+-------------------------------
                54 |               1 |                598394 |                  380 |              44873 |             0 |                0 |            4960 |                     0 |          5862 | 2024-03-28 20:07:47.401874+03
(1 row)
```

Видим 54 контрольные точки, следовательно, некоторые выполнились досрочно. Это может быть связано с тем, что при достижении max_wal_size выполняются дополнительные контрольные точки.

### Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

Посмотрим текущие настройки:
```sql
select name, setting from pg_settings where name in ('synchronous_commit');
```
В консоль выведется:
```
        name        | setting 
--------------------+---------
 synchronous_commit | on
(1 row)
```

Включим асинхронный режим:
```sql
alter SYSTEM set synchronous_commit = off;
select pg_reload_conf();
select name, setting from pg_settings where name in ('synchronous_commit');
\q
```
В консоль выведется:
```
ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)

        name        | setting 
--------------------+---------
 synchronous_commit | off
(1 row)
```

Снова подадим нагрузку (8 клиентов, вывод прогресса каждые 60 секунд, время нагрузки 600 секунд):
```
sudo -u postgres pgbench -c8 -P 60 -T 600 postgres
```

В консоль выведется:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 1823.2 tps, lat 4.334 ms stddev 2.737, 0 failed
progress: 120.0 s, 1842.7 tps, lat 4.287 ms stddev 2.786, 0 failed
progress: 180.0 s, 1832.1 tps, lat 4.315 ms stddev 2.676, 0 failed
progress: 240.0 s, 1834.0 tps, lat 4.310 ms stddev 2.724, 0 failed
progress: 300.0 s, 1918.0 tps, lat 4.118 ms stddev 2.818, 0 failed
progress: 360.0 s, 1853.8 tps, lat 4.261 ms stddev 2.818, 0 failed
progress: 420.0 s, 1842.1 tps, lat 4.292 ms stddev 2.714, 0 failed
progress: 480.0 s, 1711.2 tps, lat 4.637 ms stddev 2.740, 0 failed
progress: 540.0 s, 1696.0 tps, lat 4.678 ms stddev 2.830, 0 failed
progress: 600.0 s, 1840.3 tps, lat 4.296 ms stddev 2.735, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1091613
number of failed transactions: 0 (0.000%)
latency average = 4.347 ms
latency stddev = 2.763 ms
initial connection time = 26.342 ms
tps = 1819.386133 (without initial connection time)
```

Предыдущее значение tps: 807.408241.  
Текущее значение tps: 1819.386133.  
Т.к. в асинхронном режиме транзакции выполняются параллельно, этого следовало ожидать.

### Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

#### Создадим кластер:
```
sudo pg_createcluster 15 my_claster --start -- --data-checksums
```

В консоль выведется:
```
Creating new PostgreSQL cluster 15/my_claster ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/my_claster --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with this locale configuration:
  provider:    libc
  LC_COLLATE:  en_US.UTF-8
  LC_CTYPE:    en_US.UTF-8
  LC_MESSAGES: en_US.UTF-8
  LC_MONETARY: ru_RU.UTF-8
  LC_NUMERIC:  ru_RU.UTF-8
  LC_TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/my_claster ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster    Port Status Owner    Data directory                    Log file
15  my_claster 5433 online postgres /var/lib/postgresql/15/my_claster /var/log/postgresql/postgresql-15-my_claster.log
```

Подключимся к кластеру:  
```
sudo -u postgres psql -p 5433
```
Теперь можно вводить команды SQL.  
#### Создадим таблицу и добавим в неё несколько значений:  
```sql
create table my_table(data int);
insert into my_table (select * from generate_series(1, 100) as value);
```

В консоль выведется:
```
CREATE TABLE
INSERT 0 100
```

Определим путь до таблицы (он понадобится в дальнейшем):
```
select pg_relation_filepath('my_table');
```

В консоль выведется:
```
 pg_relation_filepath 
----------------------
 base/5/16391
(1 row)
```

#### Отключим кластер:
```
sudo -u postgres pg_ctlcluster 15 my_claster stop
```
В консоль выведется:
```
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-my_claster
```

Проверим, что кластер отключен:
```
sudo -u postgres pg_lsclusters
```

В консоль выведется:
```
Ver Cluster    Port Status Owner    Data directory                    Log file
15  main       5432 online postgres /var/lib/postgresql/15/main       /var/log/postgresql/postgresql-15-main.log
15  my_claster 5433 down   postgres /var/lib/postgresql/15/my_claster /var/log/postgresql/postgresql-15-my_claster.log
```

#### Скорректируем данные в таблице:
Заменим в редакторе nano пару байт. Путь до таблицы в кластере берём из раздела про создание таблицы:
```
sudo nano /var/lib/postgresql/15/my_claster/base/5/16391
```

#### Включим кластер и сделаем выборку из таблицы:
```
sudo -u postgres pg_ctlcluster 15 my_claster start
```
В консоль выведется:
```
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-my_claster
```

Подключимся к кластеру:  
```
sudo -u postgres psql -p 5433
```
Теперь можно вводить команды SQL.  
Делаем выборку из таблицы:
```sql
select * from my_table;
```

В консоль выведется:
```
WARNING:  page verification failed, calculated checksum 63580 but expected 26218
ERROR:  invalid page in block 0 of relation base/5/16391
```
#### Что и почему произошло?

Не совпадает контрольная сумма, а также указан некорректный блок данных в самом начале.

#### Как проигнорировать ошибку и продолжить работу?

Можно отключить проверку контрольной суммы (с перезагрузкой конфигурации):
```sql
alter SYSTEM set ignore_checksum_failure = on;
select pg_reload_conf();
```
В консоль выведется:
```
ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)
```

Снова делаем выборку из таблицы:
```sql
select * from my_table;
```

В консоль выведется:
```
data 
------
    1
    2
    3
    4
    5
    6
    7
    8
    9
   13
   11
   12
   13
   14
   15
   16
   17
... (пропустим остальные значения)
```

Как видим, были добавлены значения от 0 до 100, но после правок вместо 10 видим 13.  
Тем не менее, сама таблица загрузилась.
