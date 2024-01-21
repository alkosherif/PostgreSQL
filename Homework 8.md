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

В логах видим, что ожидание блокировки длилось более 200 мс и завершилось после 21 секунды ожидания.
