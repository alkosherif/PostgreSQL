# MVCC, vacuum и autovacuum.  Настройка autovacuum с учетом особеностей производительности
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
7.1 Выбрать Основную память 4096 МБ  
7.2 Выбрать 2 процессора  
8. Открыть раздел "Жёсткий диск"  
8.1 В разделе "Расположение и размер файла жёсткого диска" выбрать 10 ГБ  
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

Теперь подадим нагрузку:
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
\q
```

В консоль выведется:
```
 pg_current_wal_lsn | pg_current_wal_insert_lsn |   file_current_wal_lsn   | file_current_wal_insert_lsn 
--------------------+---------------------------+--------------------------+-----------------------------
 0/2262D2C8         | 0/2262D2C8                | 000000010000000000000022 | 000000010000000000000022
(1 row)
```

### Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

```
select pg_size_pretty('0/2262D2C8'::pg_lsn - '0/21CDE98'::pg_lsn) wal_size;
```

```
 wal_size 
----------
 516 MB
(1 row)
```

### Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

```sql
select * from pg_stat_bgwriter; 
```

```
 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint | buffers_clean | maxwritten_clean | buffers_backend | buffers_backend_fsync | buffers_alloc |          stats_reset          
-------------------+-----------------+-----------------------+----------------------+--------------------+---------------+------------------+-----------------+-----------------------+---------------+-------------------------------
                54 |               1 |                598394 |                  380 |              44873 |             0 |                0 |            4960 |                     0 |          5862 | 2024-03-28 20:07:47.401874+03
(1 row)
```


### Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
### Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
