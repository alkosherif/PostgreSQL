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
14.6. После этого на экране ВМ будут постоянно появляться странные артефакты, но по крайней мере заработает общий буфер обмена. При изменении размеров окна ВМ проблема пропадает.

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
progress: 10.0 s, 669.8 tps, lat 144.795 ms stddev 157.204, 0 failed
progress: 20.0 s, 686.7 tps, lat 146.244 ms stddev 162.324, 0 failed
progress: 30.0 s, 677.7 tps, lat 147.839 ms stddev 196.863, 0 failed
progress: 40.0 s, 687.9 tps, lat 145.331 ms stddev 149.798, 0 failed
progress: 50.0 s, 683.7 tps, lat 145.720 ms stddev 160.301, 0 failed
progress: 60.0 s, 698.4 tps, lat 142.335 ms stddev 152.007, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41142
number of failed transactions: 0 (0.000%)
latency average = 145.859 ms
latency stddev = 164.772 ms
initial connection time = 102.494 ms
tps = 683.906524 (without initial connection time)
```
## Настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

### Определяем рекомендуемые параметры
В этом поможет ссылка:
[https://www.pgconfig.org/#/?max_connections=50&pg_version=15&environment_name=WEB&total_ram=2&cpus=2&drive_type=HDD&arch=x86-64&os_type=linux](https://www.pgconfig.org/#/?max_connections=100&pg_version=15&environment_name=WEB&total_ram=8&cpus=2&drive_type=SSD&arch=x86-64&os_type=linux)

### Применяем рекомендуемые параметры
Редактируем в nano конфигурацию, с помощью сочетания клавиш Ctrl+W ищем параметр и меняем его:
```
sudo nano /etc/postgresql/15/main/postgresql.conf
```

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

Заходим в psql:
```
sudo -u postgres psql
```

#### Перезапуск кластера
Чтобы настройки применились, перезапустим кластер и проверим, что он запущен:
```
sudo pg_ctlcluster 15 main restart
pg_lsclusters
```
В консоль выведется:
```
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
Снова подадим нагрузку (100 клиентов, вывод прогресса каждые 10 секунд, время нагрузки 60 секунд, 4 потока):
```
sudo -u postgres pgbench -c 100 -P 10 -T 60 -j 4 postgres
```
В консоль выведется:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 653.9 tps, lat 148.525 ms stddev 161.765, 0 failed
progress: 20.0 s, 681.2 tps, lat 146.734 ms stddev 145.468, 0 failed
progress: 30.0 s, 680.1 tps, lat 147.206 ms stddev 157.316, 0 failed
progress: 40.0 s, 683.3 tps, lat 146.243 ms stddev 167.779, 0 failed
progress: 50.0 s, 687.4 tps, lat 145.068 ms stddev 159.188, 0 failed
progress: 60.0 s, 674.4 tps, lat 148.181 ms stddev 185.044, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 40703
number of failed transactions: 0 (0.000%)
latency average = 147.393 ms
latency stddev = 163.736 ms
initial connection time = 135.903 ms
tps = 676.739556 (without initial connection time)
```

tps изменилось с 683.906524 до 676.739556, то есть изменение параметров даже немного ухудшило производительность, что можно списать на погрешность.

### Попробуем поменять ещё несколько параметров
Открываем конфигурацию в nano:
```
sudo nano /etc/postgresql/15/main/postgresql.conf
```
Меняем параметры с целью уменьшения записей в журнал, включения асинхронности и улучшения оптимизации запросов:

wal_level (закомментирован) -> minimal - Пишем в журнал по минимуму
max_wal_senders (закомментирован) -> 0 - Отключаем репликацию. Не смотря на то, что репликации нет, без этого кластер не запустится при wal_level = minimal
synchronous_commit (закомментирован) -> off - Включаем асинхронный режим
full_page_writes (закомментирован) -> off; - Отключаем запись в wal на контрольных точках
fsync (закомментирован) -> off - Выключаем синхронизацию
default_statistics_target (закомментирован) -> '100'; - Улучшаем сбор статистики для лучшей оптимизации запросов

Перезагружаем кластер и смотрим, работает ли:
```
sudo pg_ctlcluster 15 main restart
pg_lsclusters
```

В консоль выведется:
```
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Ещё раз подадим нагрузку (100 клиентов, вывод прогресса каждые 10 секунд, время нагрузки 60 секунд, 4 потока):
```
sudo -u postgres pgbench -c 100 -P 10 -T 60 -j 4 postgres
```

Теперь видим:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 1250.8 tps, lat 77.791 ms stddev 111.305, 0 failed
progress: 20.0 s, 1296.8 tps, lat 77.256 ms stddev 97.208, 0 failed
progress: 30.0 s, 1263.9 tps, lat 78.576 ms stddev 100.887, 0 failed
progress: 40.0 s, 1290.5 tps, lat 78.195 ms stddev 101.205, 0 failed
progress: 50.0 s, 1283.1 tps, lat 77.736 ms stddev 98.915, 0 failed
progress: 60.0 s, 1305.2 tps, lat 76.886 ms stddev 85.218, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 77002
number of failed transactions: 0 (0.000%)
latency average = 77.827 ms
latency stddev = 99.319 ms
initial connection time = 115.370 ms
tps = 1281.830692 (without initial connection time)
```

## Написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
tps увеличилось с 676.739556 до 1281.830692 (т.е. почти в 2 раза) за счёт меньшей записи в журнал, большей асинхронности и лучшей оптимизации запросов (настройки приведены выше).
