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

1. Открываем Putty
2. В поле Host Name указываем внешний IPv4, полученный при создании ВМ (в пункте 12)
3. Нажимаем Open
4. В появившемся окне выбираем Accept
5. На запрос логина вводим логин, указанный при создании ВМ (в пункте 8)  
  
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

## 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

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

В сессии 1 запускаем транзакцию на обновление одной из строк:

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

То есть, операция в сесси 2 заблокирована и ждёт завершения первой транзакции.

Подождём секунду и завершим первую транзакцию:
```sql
SELECT pg_sleep(1);
COMMIT;
```

Во второй сесии при этом вывелось:
```sql
UPDATE 0
```

Коммитим её тоже:
```sql
COMMIT;
```

Теперь подключаемся в ещё одном окне к ВМ и проверяем, есть ли блокировки:
```
sudo tail -n 10 /var/log/postgresql/postgresql-15-main.log
```

Видим в консоли:
```
2024-01-15 21:12:06.694 UTC [4719] LOG:  checkpoint complete: wrote 950 buffers (5.8%); 0 WAL file(s) added, 0 removed, 0 recycled; write=94.796 s, sync=0.002 s, total=94.804 s; sync files=319, longest=0.002 s, average=0.001 s; distance=4314 kB, estimate=4314 kB
2024-01-15 21:12:23.100 UTC [6540] postgres@postgres LOG:  process 6540 still waiting for ShareLock on transaction 738 after 200.058 ms
2024-01-15 21:12:23.100 UTC [6540] postgres@postgres DETAIL:  Process holding the lock: 6534. Wait queue: 6540.
2024-01-15 21:12:23.100 UTC [6540] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "testdata"
2024-01-15 21:12:23.100 UTC [6540] postgres@postgres STATEMENT:  UPDATE testData SET data = 'test4' where data = 'test1';
2024-01-15 21:17:05.525 UTC [6540] postgres@postgres LOG:  process 6540 acquired ShareLock on transaction 738 after 282625.031 ms
2024-01-15 21:17:05.525 UTC [6540] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "testdata"
2024-01-15 21:17:05.525 UTC [6540] postgres@postgres STATEMENT:  UPDATE testData SET data = 'test4' where data = 'test1';
2024-01-15 21:20:31.890 UTC [4719] LOG:  checkpoint starting: time
2024-01-15 21:20:32.001 UTC [4719] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.102 s, sync=0.003 s, total=0.112 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=3883 kB
```

Нас интересует строка:
```
postgres@postgres LOG:  process 6540 acquired ShareLock on transaction 738 after 282625.031 ms
```

На её месте должно быть Access process 6540 still waiting for...
