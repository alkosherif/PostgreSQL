# Настройка PostgreSQL. Нагрузочное тестирование и тюнинг PostgreSQL	
### Развернуть виртуальную машину любым удобным способом

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

### Поставить на неё PostgreSQL 15 любым способом

В этом поможет следующий набор команд:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Появится длинный лог установки, можем его пропустить.

### Наполнение БД postgres тестовыми данными
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
done in 0.20 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.09 s, vacuum 0.03 s, primary keys 0.07 s).
```
### Проверка производительности
Подадим нагрузку (100 клиентов, вывод прогресса каждые 10 секунд, время нагрузки 60 секунд, 4 потока):
```
sudo -u postgres pgbench -c100 -P 10 -T 60 -j 4 postgres
```
В консоль выведется:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 671.1 tps, lat 144.548 ms stddev 155.088, 0 failed
progress: 20.0 s, 683.9 tps, lat 146.647 ms stddev 156.831, 0 failed
progress: 30.0 s, 692.8 tps, lat 143.938 ms stddev 156.337, 0 failed
progress: 40.0 s, 701.0 tps, lat 142.639 ms stddev 152.636, 0 failed
progress: 50.0 s, 698.7 tps, lat 143.350 ms stddev 148.833, 0 failed
progress: 60.0 s, 701.5 tps, lat 142.491 ms stddev 138.185, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41588
number of failed transactions: 0 (0.000%)
latency average = 144.249 ms
latency stddev = 151.548 ms
initial connection time = 136.808 ms
tps = 691.562576 (without initial connection time)
```
## Настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

### Определяем рекомендуемые параметры
В этом поможет ссылка:
[https://www.pgconfig.org/#/?max_connections=50&pg_version=15&environment_name=WEB&total_ram=2&cpus=2&drive_type=HDD&arch=x86-64&os_type=linux](https://www.pgconfig.org/#/?max_connections=100&pg_version=15&environment_name=WEB&total_ram=8&cpus=2&drive_type=SSD&arch=x86-64&os_type=linux)

#### Server
Этот раздел задаём сами.  
Operationg System: GNU/Linux Based  
Architecture: 64bit  
Storage type: SSD Storage  
Number of CPUs: 2  
Total Memory (GB): 8  
Max connections: 100  
Application profile: General web application  
PostgreSQL Vertion: 15  

#### Memory Configuration
shared_buffers = 2GB  
effective_cache_size = 6GB  
work_mem = 20MB  
maintenance_work_mem = 410MB  

#### Checkpoint Related Configuration
min_wal_size = 2GB  
max_wal_size = 3GB  
checkpoint_completion_target = 0.9  
wal_buffers = -1  

#### Network Related Configuration
listen_addresses = '*'  
max_connections = 100  

#### Storage Configuration
random_page_cost = 1.1  
effective_io_concurrency = 200  

#### Worker Processes Configuration
max_worker_processes = 8  
max_parallel_workers_per_gather = 2  
max_parallel_workers = 2  

### Применяем рекомендуемые параметры
Редактируем в nano конфигурацию, с помощью сочетания клавиш Ctrl+W ищем параметр и меняем его:
```
sudo nano /etc/postgresql/15/main/postgresql.conf
```
#### Memory Configuration
shared_buffers: 128MB -> 2GB  
effective_cache_size (закомментирован) -> 6GB  
work_mem (закомментирован) -> 20MB  
maintenance_work_mem (закомментирован) -> 410MB  

#### Checkpoint Related Configuration
min_wal_size = 80 MB -> 2GB  
max_wal_size = 1 GB -> 3GB  
checkpoint_completion_target (закомментирован) -> 0.9  
wal_buffers (закомментирован) -> -1  

#### Network Related Configuration
listen_addresses (закомментирован) -> '*'  
max_connections 100 -> 100  

#### Storage Configuration
random_page_cost (закомментирован) -> 1.1  
effective_io_concurrency (закомментирован) -> 200  

#### Worker Processes Configuration
max_worker_processes (закомментирован) -> 8  
max_parallel_workers_per_gather (закомментирован) -> 2  
max_parallel_workers (закомментирован) -> 2  

## нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
## написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

## Задание со *: аналогично протестировать через утилиту https://github.com/Percona-Lab/sysbench-tpcc (требует установки
https://github.com/akopytov/sysbench)


Создадим базу для тестов и запустим утилиту нагрузочного тестирования pgbench с дефолтными параметрами

sudo -u postgres psql
create database test_db;
CREATE DATABASE
Посмотрим на производительность с дефолтными параметрами

postgres=# \q

otus@otus:~$ sudo -u postgres pgbench -i test_db
otus@otus:~$ sudo -u postgres pgbench -c50 -j 2 -T 60 test_db
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 40868
number of failed transactions: 0 (0.000%)
latency average = 73.456 ms
initial connection time = 60.211 ms
tps = 680.681576 (without initial connection time)
настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

Посмотрим на рекомендации pgtune для текущей виртуалки:

https://www.pgconfig.org/#/?max_connections=50&pg_version=15&environment_name=WEB&total_ram=2&cpus=2&drive_type=HDD&arch=x86-64&os_type=linux

Linux, 64bit
CPUs: 2
RAM: 2GB
Disk: HDD
Postgres 15
Max connections 50

# Memory Configuration
shared_buffers = 512MB
effective_cache_size = 2GB
work_mem = 10MB
maintenance_work_mem = 102MB

# Checkpoint Related Configuration
min_wal_size = 2GB
max_wal_size = 3GB
checkpoint_completion_target = 0.9
wal_buffers = -1

# Network Related Configuration
listen_addresses = '*'
max_connections = 50

# Storage Configuration
random_page_cost = 4.0
effective_io_concurrency = 2

# Worker Processes Configuration
max_worker_processes = 8
max_parallel_workers_per_gather = 2
max_parallel_workers = 2

# Logging configuration for pgbadger
logging_collector = on
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
lc_messages = 'C'

# Adjust the minimum time to collect the data
log_min_duration_statement = '10s'
log_autovacuum_min_duration = 0

# CSV Configuration
log_destination = 'csvlog'

добавим эти параметры в конец файла /etc/postgresql/15/main/postgresql.conf

sudo -u postgres psql

postgres=# show config_file;
               config_file
-----------------------------------------
 /etc/postgresql/15/main/postgresql.conf
(1 row)

postgres=# \q
otus@otus:~$ sudo nano /etc/postgresql/15/main/postgresql.conf

sudo pg_ctlcluster 15 main restart
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main log/postgresql-%Y-%m-%d_%H%M%S.csv
Запустим тест

otus@otus:~$ sudo -u postgres pgbench -c50 -j 2 -T 60 test_db
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41551
number of failed transactions: 0 (0.000%)
latency average = 72.258 ms
initial connection time = 64.667 ms
tps = 691.960238 (without initial connection time)
Видим незначительный прирост производительности.

Для ускорения СУБД, поменяем еще несколько параметров, жертвуя надежностью:

sudo -u postgres psql
Режим журналирования в минимум, этого достаточно для восстановаления.

ALTER SYSTEM SET wal_level= 'minimal';
ALTER SYSTEM
Также надо поменять max_wal_senders

ALTER SYSTEM SET max_wal_senders='0';
ALTER SYSTEM
Перейдем в асинхронный режим, чтобы не ждать завершения операции.

ALTER SYSTEM SET synchronous_commit='off';
ALTER SYSTEM
Выключаем синхронизацию.

ALTER SYSTEM SET fsync='off';
ALTER SYSTEM
Отключаем запись страниц в wal при checkpoint

ALTER SYSTEM SET full_page_writes='off';
ALTER SYSTEM
Увеличение значений shared_buffers и effective_cache_size может улучшить производительность, поскольку это позволяет большему количеству данных храниться в памяти, что уменьшает количество чтения и записи на диск.

ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM
Увеличение maintenance_work_mem также может улучшить производительность при выполнении операций обслуживания базы данных, таких как ребилд индексов.

ALTER SYSTEM SET maintenance_work_mem = '256MB';
ALTER SYSTEM
Установка default_statistics_target на 100 помогает PostgreSQL лучше собирать статистику о данных и использовать ее для оптимизации запросов.

ALTER SYSTEM SET default_statistics_target = '100';
ALTER SYSTEM
Установка random_page_cost на 1.1 и effective_io_concurrency на 200 позволяет учитывать случайный доступ к данным и обрабатывать операции ввода-вывода эффективнее.

ALTER SYSTEM SET random_page_cost = '1.1';
ALTER SYSTEM
ALTER SYSTEM SET effective_io_concurrency = '200';
ALTER SYSTEM
Установка work_mem на 2048kB управляет объемом памяти, выделенной на выполнение каждого оператора сортировки или хэширования.

ALTER SYSTEM SET work_mem = '2048kB';
ALTER SYSTEM
postgres=# \q

sudo pg_ctlcluster 15 main restart
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main log/postgresql-%Y-%m-%d_%H%M%S.csv
нагрузить кластер через утилиту через утилиту pgbench

otus@otus:~$ sudo -u postgres pgbench -c50 -j 2 -T 60 test_db
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 133393
number of failed transactions: 0 (0.000%)
latency average = 22.503 ms
initial connection time = 68.622 ms
tps = 2221.913366 (without initial connection time)
написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

Удалось значительно повысить скорость - до 2220 tps со средним latency 22.503 ms. Измененные параметры с обоснованием - выше
