# Блокировки

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
12. В открывшемся окне ждём завершения создания ВМ и **сохраняем публичный IPv4 (158.160.138.140)**  

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

## Задание №1

### Задача
Настроить сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизвести ситуацию, при которой в журнале появятся такие сообщения.

### Настройка  
Проверяем, включен ли параметр log_lock_waits:
```sql
show log_lock_waits;
```

Видим, что он выключен:
```
 log_lock_waits
----------------
 off
(1 row)
```

Следовательно, **включаем log_lock_waits**:  
```
ALTER SYSTEM SET log_lock_waits = on;
```

Далее проверяем, по истечении какого минимального времени блокировка попадает в лог:
```sql
SHOW deadlock_timeout;
```

Видим, что задан таймаут 1 секунда:
```
 deadlock_timeout
------------------
 1s
(1 row)
```

**Меняем значение deadlock_timeout на 200 мс**:
```sql
ALTER SYSTEM SET deadlock_timeout = 200;
```

```
ALTER SYSTEM
```

Однако изменения значений этих переменных не достаточно, т.к. требуется перезагрузить конфигурацию (иначе, выполняя команды `SHOW log_lock_waits;` и `SHOW deadlock_timeout;` получим тот же результат).  

**Перезагружаем конфигурацию**:  
```sql
SELECT pg_reload_conf();
```

В консоль выведется:

```sql
 pg_reload_conf
----------------
 t
(1 row)
```

Теперь видим, что параметр включен:
```sql
show log_lock_waits;
```

```
 log_lock_waits
----------------
 on
(1 row)
```

Также видим, что интервал задан в 200 мс:
```sql
SHOW deadlock_timeout;
```

```
 deadlock_timeout
------------------
 200ms
(1 row)
```

### Воспроизведение блокировки

Создадим БД и таблицу:

```sql
CREATE DATABASE otus;
CREATE TABLE testData (id int, data varchar(100));
INSERT INTO testData (data) VALUES ('test1'),('test2');
SELECT * FROM testData;
```

Таблица будет выглядеть следующим образом:
```
 id | data
----+-------
    | test1
    | test2
(2 rows)
```

Теперь откроем дополнительную сессию с PostgreSQL.  
Запросим идентификатор процесса, он пригодится в дальнейшем:
```sql
select pg_backend_pid();
```

В консоли выведется:
```
 pg_backend_pid
----------------
          12605
(1 row)
```

Запоминаем число **12605**.

Теперь в сессии 1 запускаем транзакцию на обновление одной из строк:

```sql
BEGIN;
UPDATE testData SET data = 'test3' where data = 'test1';
```

В консоли выведется:
```
BEGIN
UPDATE 1
```

В сессии 2 выполняем то же самое:

```sql
BEGIN;
UPDATE testData SET data = 'test4' where data = 'test1';
```

В консоли выведется:
```
BEGIN
```

**То есть, операция в сессии 2 заблокирована и ждёт завершения первой транзакции.**

Завершим первую транзакцию:
```sql
COMMIT;
```

В сессии 2 при этом вывелось:
```sql
UPDATE 0
```

Коммитим также транзакцию сессии 2 и выходим в bash:
```sql
COMMIT;
\q
```

Теперь проверяем в сессии 2, есть ли блокировки:
```
sudo tail -n 10 /var/log/postgresql/postgresql-15-main.log
```

Видим в консоли:
```
2024-01-21 20:46:23.109 UTC [12605] postgres@postgres ERROR:  function pg_backend_id() does not exist at character 8
2024-01-21 20:46:23.109 UTC [12605] postgres@postgres HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
2024-01-21 20:46:23.109 UTC [12605] postgres@postgres STATEMENT:  select pg_backend_id();
2024-01-21 20:48:35.367 UTC [12605] postgres@postgres LOG:  process 12605 still waiting for ShareLock on transaction 738 after 200.058 ms
2024-01-21 20:48:35.367 UTC [12605] postgres@postgres DETAIL:  Process holding the lock: 12381. Wait queue: 12605.
2024-01-21 20:48:35.367 UTC [12605] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "testdata"
2024-01-21 20:48:35.367 UTC [12605] postgres@postgres STATEMENT:  update testData set data = 'test4' where data = 'test1';
2024-01-21 20:48:56.998 UTC [12605] postgres@postgres LOG:  process 12605 acquired ShareLock on transaction 738 after 21831.039 ms
2024-01-21 20:48:56.998 UTC [12605] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "testdata"
2024-01-21 20:48:56.998 UTC [12605] postgres@postgres STATEMENT:  update testData set data = 'test4' where data = 'test1';
```

Нас интересуют строки:
```
2024-01-21 20:48:35.367 UTC [12605] postgres@postgres LOG:  process 12605 still waiting for ShareLock on transaction 738 after 200.058 ms
2024-01-21 20:48:56.998 UTC [12605] postgres@postgres LOG:  process 12605 acquired ShareLock on transaction 738 after 21831.039 ms
```

В логах видим, что ожидание блокировки процессом с ранее запомненным идентификатором **12605** длилось более 200 мс и завершилось после 21 секунды ожидания.

## Задание №2

### Задача
Смоделировать ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучить возникшие блокировки в представлении pg_locks и убедиться, что все они понятны. Прислать список блокировок и объяснить, что значит каждая.

### Создание таблицы

**Сессия 1.**

Создадим таблицу с первичным ключом.
```sql
CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
```

Добавим данные:
```sql
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00),(3,3000.00);
```
Проверим, что данные добавлены:
```sql
SELECT * FROM accounts;
```

```
 acc_no | amount
--------+---------
      1 | 1000.00
      2 | 2000.00
      3 | 3000.00
(3 rows)
```

### Моделирование ситуации

Откроем транзакции в трёх разных сессиями PostgreSQL и посмотрим их идентификаторы:
```sql
BEGIN;
SELECT pg_backend_pid();
```

В каждой консоли увидим свой идентификатор:
```
 pg_backend_pid
----------------
          12780
(1 row)
```

Сессия 1: 12780  
Сессия 2: 12831  
Сессия 3: 12928  
  
Посмотрим в **сессии 1**, какие блокировки удерживает только что начавшаяся транзакция, через pg_locks, подставив идентификатор в запрос:
```sql
SELECT locktype, relation::REGCLASS, virtualxid as virtxid, transactionid as xid, mode, granted FROM pg_locks WHERE pid = 12780;
```

В консоли видим:
```
  locktype  | relation | virtxid | xid |      mode       | granted
------------+----------+---------+-----+-----------------+---------
 relation   | pg_locks |         |     | AccessShareLock | t
 virtualxid |          | 3/68    |     | ExclusiveLock   | t
(2 rows)
```

Транзакция удерживает эксклюзивную блокировку (ExclusiveLock) своего номера (locktype = virtualxid).  
А также лёгкую блокировку (AccessShareLock) отношения pg_locks, к которому мы только что обращались.  
  
Попробуем обновить строку таблицы в **сессии 1**:
```sql
UPDATE accounts SET amount = amount + 100 where acc_no = 1;
```

Смотрим снова список блокировок:
```sql
SELECT locktype, relation::REGCLASS, virtualxid as virtxid, transactionid as xid, mode, granted FROM pg_locks WHERE pid = 12780;
```

Видим уже знакомые блокировки виртуального номера транзакции (ExclusiveLock virtualxid) и отношения (AccessShareLock relation pg_locks).  
Также, видим построчные блокировки RowExclusiveLock ключа (accounts_pkey) и строки (accounts), т.к. в изменяем строку в данный момент.  
И, наконец, видим эксклюзивную блокировку номера транзакции (ExclusiveLock), т.к. транзакция ещё выполняется.  

```
   locktype    |   relation    | virtxid | xid |       mode       | granted
---------------+---------------+---------+-----+------------------+---------
 relation      | accounts_pkey |         |     | RowExclusiveLock | t
 relation      | accounts      |         |     | RowExclusiveLock | t
 relation      | pg_locks      |         |     | AccessShareLock  | t
 virtualxid    |               | 3/68    |     | ExclusiveLock    | t
 transactionid |               |         | 749 | ExclusiveLock    | t
(5 rows)
```

Обращаем внимание на строку с transactionid и значением **749**, этот идентификатор пригодится в дальнейшем.

Теперь выполним ту же команду обновления в **сессии 2**:
```sql
UPDATE accounts SET amount = amount + 100 where acc_no = 1;
```
Она зависнет, т.к. ожидает блокировки.  
При запросе таблицы блокировок в **сессии 1** увидим, что ничего не изменилось.  
Посмотрим, на таблицу блокировок для **сессии 2** в **сессии 1**, передав идентификатор **12831 сессии 2**:
```sql
SELECT locktype, relation::REGCLASS, virtualxid as virtxid, transactionid as xid, mode, granted FROM pg_locks WHERE pid = 12831;
```

Видим в консоли:
```
   locktype    |   relation    | virtxid | xid |       mode       | granted
---------------+---------------+---------+-----+------------------+---------
 relation      | accounts_pkey |         |     | RowExclusiveLock | t
 relation      | accounts      |         |     | RowExclusiveLock | t
 virtualxid    |               | 4/186   |     | ExclusiveLock    | t
 transactionid |               |         | 749 | ShareLock        | f
 tuple         | accounts      |         |     | ExclusiveLock    | t
 transactionid |               |         | 750 | ExclusiveLock    | t
(6 rows)
```

**Общее в двух сессиях**:  
1. Получена эксклюзивная блокировка виртуального номера транзакции ExclusiveLock virtualxid;  
2. Получена эксклюзивная блокировка номера транзакции ExclusiveLock transactionid;  
3. В обеих сессиях получена построчная блокировка RowExclusiveLock relation ключа и строки accounts и accounts_pkey, поскольку изменяется одна строка.  

**Отличия**:  
1. В таблице для **сессии 2** отсутствует блокировка отношения AccessShareLock relation pg_locks, так как запрос pg_locks выполняется в **сессии 1**, а не в **сессии 2**;  
2. В **сессии 2** ожидается ShareLock транзакции **сессии 1** с номером **749**, т.к. транзакция **сессии 1**  не завершена;
3. Добавился ExclusiveLock на tuple accounts, т.к. транзакция засыпает и ожидает ShareLock **транзакции 1**, а последующие транзакции, изменяющие ту же строку, будут ссылаться не на **транзакцию 2**, а на её tuple.

Наконец, выполним ту же команду обновления в **сессии 3**:
```sql
UPDATE accounts SET amount = amount + 100 where acc_no = 1;
```
Она также зависнет, т.к. ожидает блокировки.  
Посмотрим, на таблицу блокировок для **сессии 3** в **сессии 1**, передав идентификатор **12928 сессии 3**:
```sql
SELECT locktype, relation::REGCLASS, virtualxid as virtxid, transactionid as xid, mode, granted FROM pg_locks WHERE pid = 12928;
```

В консоли видим:
```
   locktype    |   relation    | virtxid | xid |       mode       | granted
---------------+---------------+---------+-----+------------------+---------
 relation      | accounts_pkey |         |     | RowExclusiveLock | t
 relation      | accounts      |         |     | RowExclusiveLock | t
 virtualxid    |               | 5/71    |     | ExclusiveLock    | t
 transactionid |               |         | 751 | ExclusiveLock    | t
 tuple         | accounts      |         |     | ExclusiveLock    | f
(5 rows)
```

В отличие от блокировок **сессии 2**, в таблице для **сессии 3** отсутствует ShareLock transactionid 750, так как вместо ссылки на идентификатор транзакции используется ссылка на tuple для ожидания завершения транзакции **сессии 2**. Если открыть ещё одну сессию, таблица блокировок в ней будет выглядеть также (поменяются только идентификаторы текущей транзакции).

## Задание №3

### Задача
Воспроизвести взаимоблокировку трех транзакций. Определить, можно ли разобраться в ситуации постфактум, изучая журнал сообщений.

### Моделирование ситуации

На вебинаре рассматривался случай для двух транзакций, когда первая ожидает завершения второй, а вторая - завершения первой.  
По аналогии, для воспроизведения ситуации с тремя транзакции нужно, чтобы 3я транзакция ожидала завершения 2й, 2я - 1й, а 1я - 3й.  
Открываем три сессии PostgreSQL.  

Шаг 1. Сессия 1:
```sql
BEGIN;
UPDATE accounts SET amount = amount + 100 where acc_no = 1;
```

В консоли выведется:
```
BEGIN
UPDATE 1
```

Шаг 2. Сессия 2:
```sql
BEGIN;
UPDATE accounts SET amount = amount + 100 where acc_no = 2;
```

В консоли выведется:
```
BEGIN
UPDATE 1
```

Шаг 3. Сессия 3:
```sql
BEGIN;
UPDATE accounts SET amount = amount + 100 where acc_no = 3;
```

В консоли выведется:
```
BEGIN
UPDATE 1
```

Шаг 4. Сессия 1:
```sql
UPDATE accounts SET amount = amount + 100 where acc_no = 3;
```

В консоли ничего не выведется, сессия зависнет.

Шаг 5. Сессия 2:
```sql
UPDATE accounts SET amount = amount + 100 where acc_no = 1;
```

В консоли ничего не выведется, сессия зависнет.

Шаг 6. Сессия 3:
```sql
UPDATE accounts SET amount = amount + 100 where acc_no = 2;
```

В консоли выведется ошибка.
