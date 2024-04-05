# Хранимые функции и процедуры. Триггеры, поддержка заполнения витрин

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

### Установка PostgreSQL 15

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

### Загрузка тестовой базы

Выполним скрипт hw_triggers.sql:
```sql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ;

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```

В консоль выведется:
```
DROP SCHEMA
CREATE SCHEMA
SET
CREATE TABLE
INSERT 0 2
CREATE TABLE
INSERT 0 4
        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
CREATE TABLE
```

### Актуализируем таблицу good_sum_mart

Таблица пуста, перенесём в неё данные:
```sql
insert into good_sum_mart(good_name, sum_sale)
    select g.good_name, sum(g.good_price * s.sales_qty) from goods g
    inner join sales s
        on s.good_id = g.goods_id
    group by g.good_name;
```
В консоль выведется:
```
INSERT 0 2
```
Посмотрим, что лежит в таблице:
```sql
select * from good_sum_mart;
```

В консоль выведется:
```
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

Полученные данные в таблице совпадают с запрошенным ранее отчётом.

### Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

Создадим функцию, расчитывающую новую сумму при изменении таблицы sales и добавим триггер:
```sql
create or replace function update_sum() returns trigger as
$$
declare
    modifying_good_name varchar(63);
    good_price_old numeric(12, 2);
    good_price_new numeric(12, 2);
    sum_increment numeric(16, 2);
begin
    sum_increment := 0;
   
    if (old is not null) then
        select good_name, good_price into modifying_good_name, good_price_old from pract_functions.goods where goods_id = old.good_id;
    end if;

    if (new is not null) then
        select good_name, good_price into modifying_good_name, good_price_new from pract_functions.goods where goods_id = new.good_id;
    end if; 

    select
        case
            when tg_op = 'INSERT' then new.sales_qty * good_price_new
            when tg_op = 'DELETE' then -old.sales_qty * good_price_old
            when tg_op = 'UPDATE' then (new.sales_qty * good_price_new) - (old.sales_qty * good_price_old)
            else 0
        end
    into sum_increment;

    if exists (select 1 from pract_functions.good_sum_mart where good_name = modifying_good_name)
    then 
        update pract_functions.good_sum_mart
        set sum_sale = pract_functions.good_sum_mart.sum_sale + sum_increment
        where good_name = modifying_good_name;
    else
        insert into pract_functions.good_sum_mart (good_name, sum_sale)
        values (modifying_good_name, sum_increment);
    end if;  
  return new;
end;
$$
language plpgsql;

-- Добавляем триггер:
create trigger trigger_sales
after insert or delete or update on pract_functions.sales
for each row execute function update_sum();
```

В консоль выведется:
```
CREATE FUNCTION
CREATE TRIGGER
```

Добавим запись о продаже автомобиля и посмотрим, как изменилась витрина:
```sql
insert into sales (good_id, sales_qty) values (2, 1);
select * from good_sum_mart;
```
В консоль выведется:
```
INSERT 0 1
        good_name         |   sum_sale   
--------------------------+--------------
 Спички хозайственные     |        65.50
 Автомобиль Ferrari FXX K | 370000000.02
(2 rows)
```
Как видно, сумма продаж автомобиля увеличилась вдвое.

Попробуем удалить все записи о продаже автомобилей и снова посмотрим на витрину:
```sql
delete from sales where good_id = 2;
select * from good_sum_mart;
```
В консоль выведется:
```
DELETE 2
        good_name         | sum_sale 
--------------------------+----------
 Спички хозайственные     |    65.50
 Автомобиль Ferrari FXX K |     0.00
(2 rows)
```
Видим, что запись об автомобиле осталась, но сумма продаж изменилась на 0.  
Если нужно, чтобы строки без продаж не выводились, можно добавить в функцию update_sum перед return new удаление записей с нулевой суммой:
```sql
delete from pract_functions.good_sum_mart where sum_sale = 0;
```
Т.к. продаж с отрицательной суммой и нулевой суммой, предположитель, не будет, такой подход должен сработать.  
Не будем менять функцию, проверим, как работает обновление строки продаж:
```sql
update sales set sales_qty = 12 where sales_qty = 120;
select * from good_sum_mart;
```
В консоль выведется:
```
UPDATE 1
        good_name         | sum_sale 
--------------------------+----------
 Автомобиль Ferrari FXX K |     0.00
 Спички хозайственные     |    11.50
(2 rows)
```
Т.е. уменьшили число проданных спичек со 120 до 12 (на 108), сумма изменилась с 65.50 до 11.50 (на 54). Цена одной позиции спичек равна 108 / 54 = 2. А должна быть 0.5...

### Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.
