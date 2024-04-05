# Виды и устройство репликации в PostgreSQL. Практика применения

## Подготовка

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
14.6. После этого на экране ВМ будут постоянно появляться странные артефакты, но по крайней мере заработает общий буфер обмена. При изменении размеров окна ВМ проблема пропадает.

### Клонирование ВМ

Т.к. для выполнения ДЗ потребуется несколько ВМ, можно не проделывать все шаги заново, а просто:
* нажать правой кнопкой по ВМ
* выбрать "Клонировать"
* в появившемся окне заполнить имя ВМ
* нажимать далее, пока не создастся копия ВМ

### Настройка сети

С настройками по умолчанию у всех ВМ будет одинаковый IP адрес, что будет мешать при настройке репликации.  
Чтобы этого избежать, нужно:
* Открыть настройки (Машина > Настроить)
* Перейти в раздел "Сеть"
* Выбрать тип подключения "Сетевой мост"

### Установка PostgreSQL 15

На каждой из трёх ВМ установим PostgreSQL.  
В этом поможет следующий набор команд:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Появится длинный лог установки, можем его пропустить.

Заходим в psql:
```
sudo -u postgres psql
```

В консоль выведется:
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```

Теперь можно вводить команды SQL.

## Выполнение ДЗ

### Настройка ВМ для репликации

**На каждой ВМ** укажем прослушивание любых адресов:
```
sudo -u postgres psql -c "alter system set listen_addresses = '*'"
sudo systemctl restart postgresql@15-main
```
В консоль выведется:
```
ALTER SYSTEM
```

Посмотрим ip адреса каждой ВМ командой ifconfig.  
Если ifconfig не установлен, можно это исправить следующей командой (она и так выведется в подсказке при вызове ifconfig):
```
sudo apt install net-tools
```
ВМ 1: 192.168.31.6  
ВМ 2: 192.168.31.7  
ВМ 3: 192.168.31.8  

Теперь нужно на каждой ВМ прописать адреса других ВМ.  
Открываем nano и редактируем конфигурацию:
```
sudo nano /etc/postgresql/15/main/pg_hba.conf
```

Добавляем адреса на **ВМ 1**:
```
host          postgres  postgres  192.168.31.7/32  trust
host          postgres  postgres  192.168.31.8/32  trust
```

Добавляем адреса на **ВМ 2**:
```
host          postgres  postgres  192.168.31.6/32  trust
host          postgres  postgres  192.168.31.8/32  trust
```

Добавляем адреса на **ВМ 3**:
```
host          postgres  postgres  192.168.31.6/32  trust
host          postgres  postgres  192.168.31.7/32  trust
```

На **каждой ВМ** перезагружаем конфигурацию:
```
sudo -u postgres psql -c 'select pg_reload_conf();'
```
В консоль выведется:
```
 pg_reload_conf 
----------------
 t
(1 row)
```

Также на каждой ВМ задаём логический wal_level и перезапускаем кластер:
```
sudo -u postgres psql -c 'alter system set wal_level = logical;'
sudo systemctl restart postgresql@15-main
sudo -u postgres psql -c 'show wal_level;'
```
В консоль выведется:
```
ALTER SYSTEM
 wal_level 
-----------
 logical
(1 row)
```

### На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.

Заходим в psql на **ВМ 1**:
```
sudo -u postgres psql
```
В консоль выведется:
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```
Теперь можно вводить SQL-команды.  
Создадим таблицы:
```sql
create table test (name varchar primary key);
create table test2 (name varchar primary key);
```
В консоль выведется:
```
CREATE TABLE
CREATE TABLE
```

### Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.

Продолжаем работать на **ВМ 1**. Создадим публицацию:
```sql
create publication test_publication_vm1 for table test;
```
В консоль выведется:
```
CREATE PUBLICATION
```

Подпишемся на публикацию таблицы test 2 с ВМ 2 (всё ещё остаёмся на **ВМ 1**):
```sql
create subscription test2_subscription_vm1_from_vm2 connection 'host=192.168.31.7 user=postgres dbname=postgres' publication test2_publication_vm2 with (copy_data = true);
```
В консоль выведется:
```
WARNING:  publication "test2_publication_vm2" does not exist on the publisher
NOTICE:  created replication slot "test2_subscription_vm1_from_vm2" on publisher
CREATE SUBSCRIPTION
```

### На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.

Заходим в psql на **ВМ 2**:
```
sudo -u postgres psql
```
В консоль выведется:
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```
Теперь можно вводить SQL-команды.  
Создадим таблицы:
```sql
create table test (name varchar primary key);
create table test2 (name varchar primary key);
```
В консоль выведется:
```
CREATE TABLE
CREATE TABLE
```

### Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.

Продолжаем работать на **ВМ 2**. Создадим публицацию:
```sql
create publication test2_publication_vm2 for table test2;
```
В консоль выведется:
```
CREATE PUBLICATION
```

Подпишемся на публикацию таблицы test с ВМ 1 (всё ещё остаёмся на **ВМ 2**):
```sql
create subscription test2_subscription_vm2_from_vm1 connection 'host=192.168.31.6 user=postgres dbname=postgres' publication test_publication_vm1 with (copy_data = true);
```
В консоль выведется:
```
NOTICE:  created replication slot "test2_subscription_vm2_from_vm1" on publisher
CREATE SUBSCRIPTION
```

### 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).

Заходим в psql на **ВМ 3**:
```
sudo -u postgres psql
```
В консоль выведется:
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```
Теперь можно вводить SQL-команды.  
Создадим таблицы:
```sql
create table test (name varchar primary key);
create table test2 (name varchar primary key);
```
В консоль выведется:
```
CREATE TABLE
CREATE TABLE
```

Подпишемся на публикацию таблицы test 2 с ВМ 2 (всё ещё остаёмся на **ВМ 3**):
```sql
create subscription test2_subscription_vm3_from_vm2 connection 'host=192.168.31.7 user=postgres dbname=postgres' publication test2_publication_vm2 with (copy_data = true);
```
В консоль выведется:
```
NOTICE:  created replication slot "test2_subscription_vm3_from_vm2" on publisher
CREATE SUBSCRIPTION
```

Подпишемся на публикацию таблицы test с ВМ 1 (всё ещё остаёмся на **ВМ 3**):
```sql
create subscription test2_subscription_vm3_from_vm1 connection 'host=192.168.31.6 user=postgres dbname=postgres' publication test_publication_vm1 with (copy_data = true);
```
В консоль выведется:
```
NOTICE:  created replication slot "test2_subscription_vm3_from_vm1" on publisher
CREATE SUBSCRIPTION
```

### Проверка работы репликации

#### Проверка изменения таблицы test

На **ВМ 1** добавляем данные:
```sql
insert into test values ('123'), ('4567');
```
В консоль выведется:
```
INSERT 0 2
```

На **ВМ 2** и **ВМ 3** смотрим, подтянулись ли данные:
```sql
select * from test;
```

На обеих ВМ в консоль выведется:
```
 name 
------
 123
 4567
(2 rows)
```

#### Проверка изменения таблицы test2

На **ВМ 2** добавляем данные:
```sql
insert into test2 values ('hello'), ('world');
```
В консоль выведется:
```
INSERT 0 2
```

На **ВМ 1** и **ВМ 3** смотрим, подтянулись ли данные:
```sql
select * from test2;
```

На **ВМ 1** в консоль выведется:
```
 name 
------
(0 rows)
```

На **ВМ 3** в консоль выведется:
```
 name  
-------
 hello
 world
(2 rows)
```

Видим, что на ВМ, которая создавалась до публикации, данные не подтянулись, а на ВМ, которая создавалась после - подтянулись.  
Попробуем пересоздать подписку на **ВМ 1**:
```
drop subscription test2_subscription_vm1_from_vm2;
create subscription test2_subscription_vm1_from_vm2 connection 'host=192.168.31.7 user=postgres dbname=postgres' publication test2_publication_vm2 with (copy_data = true);
```

В консоль выведется:
```
NOTICE:  dropped replication slot "test2_subscription_vm1_from_vm2" on publisher
DROP SUBSCRIPTION
NOTICE:  created replication slot "test2_subscription_vm1_from_vm2" on publisher
CREATE SUBSCRIPTION
```

Посмотрим, что сейчас в таблице test2 на **ВМ 1**:
```sql
select * from test2;
```
В консоль выведется:
```
 name  
-------
 hello
 world
(2 rows)
```

Попробуем добавить ещё одну строку на **ВМ 2**:
```sql
insert into test2 values ('new value');
```
В консоль выведется:
```
INSERT 0 1
```

Снова посмотрим, что сейчас в таблицах test2 на **ВМ 1** и **ВМ 3**:
```sql
select * from test2;
```
В консоль на обеих ВМ выведется:
```
   name    
-----------
 hello
 world
 new value
(3 rows)
```

Теперь репликация работает на всех ВМ.
