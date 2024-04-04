# Сбор и использование статистики. Работа с join'ами, статистикой 	

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

### К работе приложить структуру таблиц, для которых выполнялись соединения

#### Создание таблиц

Будем использовать следующие таблицы:
* Предметы (details)
* Цвет (colors)
* Форма (shapes)

```sql
create table colors (
    Id serial primary key,
    Name varchar(100) not null,
    Code varchar(10) not null
);

create table shapes (
    Id serial primary key,
    Name varchar(100) not null
);

create table details (
    Id serial primary key,
    Description varchar (100) not null,
    ColorId int null,
    ShapeId int null,
    constraint fk_color
      foreign key(ColorId) 
        references colors(Id),
    constraint fk_shape
      foreign key(ShapeId) 
        references shapes(Id)
);
```

В консоли на каждую таблицу выведется:
```
CREATE TABLE
```

#### Добавим данные

```sql
-- Заполним таблицу цветов:
insert into colors (Name, Code) values
('Red', 'FF0000'),     -- 1
('Green', '00FF00'),   -- 2
('Blue', '0000FF'),    -- 3
('Yellow', 'FFFF00'),  -- 4
('Cyan', '00FFFF'),    -- 5
('Pink', 'FFCDCB'),    -- 6
('White', 'FFFFFF'),   -- 7
('Black', '000000'),   -- 8
('Brown', 'A52A2A'),   -- 9
('Gray', '808080'),    -- 10
('Tomato', 'FF6347'),  -- 11
('Lime', '1E90FF'),    -- 12
('Orange', 'F08080'),  -- 13
('Gold', 'DEB887')     -- 14
;

-- Добавим фигуры:
insert into shapes (Name) values
('Circle'),        -- 1
('Square'),        -- 2
('Triangle'),      -- 3
('Rectangle'),     -- 4
('Cube'),          -- 5
('Cylinder'),      -- 6
('Parallelogram'), -- 7
('Trapezium'),     -- 8
('Cone'),          -- 9
('Octagon'),       -- 10
('Hexagon'),       -- 11
('Sphere')         -- 12
;

-- Добавим детали:
insert into details (Description, ColorId, ShapeId) values
('Красный круг', 1, 1),
('Чёрный квадрат', 8, 4),
('Деталь', 3, null),
('Светодиод', 1, null),
('Треугольник', null, 3),
('Хоровод', null, 1),
('Синий куб', 3, 5),
('Зелёная трапеция', 2, 8)
;
```

В консоль выведется:
```
INSERT 0 14
INSERT 0 12
INSERT 0 8
```

### Реализовать прямое соединение двух или более таблиц

Запросим объекты с известной формой и цветом:
```sql
select d.Description, c.Name as Color, s.Name as Shape from details d
join colors c
  on d.ColorId = c.Id
join shapes s
  on d.ShapeId = s.Id;
```

В консоль выведется:
```
   description    | color |   shape   
------------------+-------+-----------
 Красный круг     | Red   | Circle
 Чёрный квадрат   | Black | Rectangle
 Синий куб        | Blue  | Cube
 Зелёная трапеция | Green | Trapezium
(4 rows)
```

Посмотрим на план запроса:
```sql
explain select d.Description, c.Name as Color, s.Name as Shape from details d
join colors c
  on d.ColorId = c.Id
join shapes s
  on d.ShapeId = s.Id;
```

В консоль выведется:
```
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Hash Join  (cost=33.50..48.25 rows=310 width=654)
   Hash Cond: (d.shapeid = s.id)
   ->  Hash Join  (cost=16.30..30.22 rows=310 width=440)
         Hash Cond: (d.colorid = c.id)
         ->  Seq Scan on details d  (cost=0.00..13.10 rows=310 width=226)
         ->  Hash  (cost=12.80..12.80 rows=280 width=222)
               ->  Seq Scan on colors c  (cost=0.00..12.80 rows=280 width=222)
   ->  Hash  (cost=13.20..13.20 rows=320 width=222)
         ->  Seq Scan on shapes s  (cost=0.00..13.20 rows=320 width=222)
(9 rows)
```

Видим, что используется достаточно быстрый Hash Join, но внутри всё равно есть Seq Scan.  

Посмотрим, что будет, если вызвать:
```sql
analyse shapes;
analyse details;
analyse colors;
```
В консоль выведется ANALYZE на каждую строку.
Снова вызовем тот же explain, на этот раз результат будет таким:
```
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Hash Join  (cost=2.59..3.71 rows=8 width=34)
   Hash Cond: (i.shapeid = s.id)
   ->  Hash Join  (cost=1.32..2.42 rows=8 width=30)
         Hash Cond: (i.colorid = c.id)
         ->  Seq Scan on details i  (cost=0.00..1.08 rows=8 width=29)
         ->  Hash  (cost=1.14..1.14 rows=14 width=9)
               ->  Seq Scan on colors c  (cost=0.00..1.14 rows=14 width=9)
   ->  Hash  (cost=1.12..1.12 rows=12 width=12)
         ->  Seq Scan on shapes s  (cost=0.00..1.12 rows=12 width=12)
(9 rows)
```

Видим, что методы поиска остались теми же, но количество сканируемых строк значительно уменьшилось, а общая стоимость cost уменьшилась с 48.25 до 3.71, т.е. примерно в 13 раз.

### Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

Запросим все объекты:
```sql
select d.Description, c.Name as Color, s.Name as Shape from details d
left join colors c
  on d.ColorId = c.Id
left join shapes s
  on d.ShapeId = s.Id;
```

В консоль выведется:
```
      description       | color |   shape   
------------------------+-------+-----------
 Красный круг           | Red   | Circle
 Чёрный квадрат         | Black | Rectangle
 Море                   | Blue  | 
 Тюльпан                | Blue  | 
 Бермудский треугольник |       | Triangle
 Хоровод                |       | Circle
 Синий куб              | Blue  | Cube
 Зелёная трапеция       | Green | Trapezium
(8 rows)
```

Посмотрим на план запроса:
```sql
explain select d.Description, c.Name as Color, s.Name as Shape from details d
left join colors c
  on d.ColorId = c.Id
left join shapes s
  on d.ShapeId = s.Id;
```

Без analyse в консоль выведется:
```
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Hash Left Join  (cost=33.50..48.25 rows=310 width=654)
   Hash Cond: (d.shapeid = s.id)
   ->  Hash Left Join  (cost=16.30..30.22 rows=310 width=440)
         Hash Cond: (d.colorid = c.id)
         ->  Seq Scan on details d  (cost=0.00..13.10 rows=310 width=226)
         ->  Hash  (cost=12.80..12.80 rows=280 width=222)
               ->  Seq Scan on colors c  (cost=0.00..12.80 rows=280 width=222)
   ->  Hash  (cost=13.20..13.20 rows=320 width=222)
         ->  Seq Scan on shapes s  (cost=0.00..13.20 rows=320 width=222)
(9 rows)
```

Снова используется быстрый метод сканирования  с некоторыми обращениями к Seq Scan.  

С analyse по запросу explain в консоль выведется:
```
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Hash Left Join  (cost=2.59..3.71 rows=8 width=34)
   Hash Cond: (d.shapeid = s.id)
   ->  Hash Left Join  (cost=1.32..2.42 rows=8 width=30)
         Hash Cond: (d.colorid = c.id)
         ->  Seq Scan on details d  (cost=0.00..1.08 rows=8 width=29)
         ->  Hash  (cost=1.14..1.14 rows=14 width=9)
               ->  Seq Scan on colors c  (cost=0.00..1.14 rows=14 width=9)
   ->  Hash  (cost=1.12..1.12 rows=12 width=12)
         ->  Seq Scan on shapes s  (cost=0.00..1.12 rows=12 width=12)
(9 rows)
```

Как и в прошлый раз, уменьшилось количество сканируемых строк, но план остался тем же.

### Реализовать кросс соединение двух или более таблиц

Посмотрим все возможные сочетания цветов и фигур:
```sql
select c.Name as Color, s.Name as Shape from colors c cross join shapes s;
```
В консоль выведется (сокращено):
```
 color  |     shape     
--------+---------------
 Red    | Circle
 Green  | Circle
 Blue   | Circle
...
 Red    | Square
 Green  | Square
...
 Gray   | Octagon
 Tomato | Octagon
...
 Orange | Sphere
 Gold   | Sphere
(168 rows)
```

Посмотрим на план запроса:
```sql
explain select c.Name as Color, s.Name as Shape from colors c cross join shapes s;
```
В консоль без analyse выведется:
```
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Nested Loop  (cost=0.00..1146.70 rows=89600 width=436)
   ->  Seq Scan on shapes s  (cost=0.00..13.20 rows=320 width=218)
   ->  Materialize  (cost=0.00..14.20 rows=280 width=218)
         ->  Seq Scan on colors c  (cost=0.00..12.80 rows=280 width=218)
(4 rows)
```

Используется довольно дорогой Nested Loop.

Посмотрим на результаты с analyse:
```
                             QUERY PLAN                              
---------------------------------------------------------------------
 Nested Loop  (cost=0.00..4.39 rows=168 width=13)
   ->  Seq Scan on colors c  (cost=0.00..1.14 rows=14 width=5)
   ->  Materialize  (cost=0.00..1.18 rows=12 width=8)
         ->  Seq Scan on shapes s  (cost=0.00..1.12 rows=12 width=8)
(4 rows)
```

Всё ещё nested loop, но общий cost уменьшился с 1146.70 до 4.39, т.е. примерно в 261 раз!

### Реализовать полное соединение двух или более таблиц
Посмотрим, у каких деталей нет формы и каких форм детали отсутствуют:
```sql
select d.Description as Detail, s.Name as Shape from details d full join shapes s on d.ShapeId = s.Id;
```
В консоль выведется:
```
      detail      |     shape     
------------------+---------------
 Красный круг     | Circle
 Чёрный квадрат   | Rectangle
 Деталь           | 
 Светодиод        | 
 Треугольник      | Triangle
 Хоровод          | Circle
 Синий куб        | Cube
 Зелёная трапеция | Trapezium
                  | Hexagon
                  | Sphere
                  | Octagon
                  | Square
                  | Cylinder
                  | Cone
                  | Parallelogram
(15 rows)
```

Посмотрим план запроса:
```sql
explain select d.Description as Detail, s.Name as Shape from details d full join shapes s on d.ShapeId = s.Id;
```

В консоль без analyse выведется:
```
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Hash Full Join  (cost=17.20..31.12 rows=320 width=436)
   Hash Cond: (d.shapeid = s.id)
   ->  Seq Scan on details d  (cost=0.00..13.10 rows=310 width=222)
   ->  Hash  (cost=13.20..13.20 rows=320 width=222)
         ->  Seq Scan on shapes s  (cost=0.00..13.20 rows=320 width=222)
(5 rows)
```

Здесь также используется сканирование по хэшу.  

С analyse:
```
                              QUERY PLAN                              
----------------------------------------------------------------------
 Hash Full Join  (cost=1.27..2.37 rows=12 width=29)
   Hash Cond: (d.shapeid = s.id)
   ->  Seq Scan on details d  (cost=0.00..1.08 rows=8 width=25)
   ->  Hash  (cost=1.12..1.12 rows=12 width=12)
         ->  Seq Scan on shapes s  (cost=0.00..1.12 rows=12 width=12)
(5 rows)
```

Как видим, план всегда остаётся тем же, но количество сканируемых строк значительно уменьшается, как и общая цена запроса.  

### Реализовать запрос, в котором будут использованы разные типы соединений

Запросим объекты с чётко заданной формой и их цвета (при наличии):
```sql
select d.Description, c.Name as Color, s.Name as Shape from details d
left join colors c
  on d.ColorId = c.Id
join shapes s
  on d.ShapeId = s.Id;
```

В консоль выведется:
```
   description    | color |   shape   
------------------+-------+-----------
 Красный круг     | Red   | Circle
 Чёрный квадрат   | Black | Rectangle
 Треугольник      |       | Triangle
 Хоровод          |       | Circle
 Синий куб        | Blue  | Cube
 Зелёная трапеция | Green | Trapezium
(6 rows)
```

Посмотрим на план запроса:
```sql
explain select d.Description, c.Name as Color, s.Name as Shape from details d
left join colors c
  on d.ColorId = c.Id
join shapes s
  on d.ShapeId = s.Id;
```

В консоль выведется:
```
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Hash Join  (cost=33.50..48.25 rows=310 width=654)
   Hash Cond: (d.shapeid = s.id)
   ->  Hash Left Join  (cost=16.30..30.22 rows=310 width=440)
         Hash Cond: (d.colorid = c.id)
         ->  Seq Scan on details d  (cost=0.00..13.10 rows=310 width=226)
         ->  Hash  (cost=12.80..12.80 rows=280 width=222)
               ->  Seq Scan on colors c  (cost=0.00..12.80 rows=280 width=222)
   ->  Hash  (cost=13.20..13.20 rows=320 width=222)
         ->  Seq Scan on shapes s  (cost=0.00..13.20 rows=320 width=222)
(9 rows)
```

Снова достаточно оптимальное сканирование по хэшу.  

С analyse:
```
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Hash Join  (cost=2.59..3.71 rows=8 width=34)
   Hash Cond: (d.shapeid = s.id)
   ->  Hash Left Join  (cost=1.32..2.42 rows=8 width=30)
         Hash Cond: (d.colorid = c.id)
         ->  Seq Scan on details d  (cost=0.00..1.08 rows=8 width=29)
         ->  Hash  (cost=1.14..1.14 rows=14 width=9)
               ->  Seq Scan on colors c  (cost=0.00..1.14 rows=14 width=9)
   ->  Hash  (cost=1.12..1.12 rows=12 width=12)
         ->  Seq Scan on shapes s  (cost=0.00..1.12 rows=12 width=12)
(9 rows)
```

Результаты те же, что и раньше.

### Сделать комментарии на каждый запрос

Оставлены комментарии в каждом пункте. Резюмируя - с analyse план запроса не меняется, но сами select'ы происходят намного оптимальнее за счёт меньшего числа сканируемых строк. Особенно это заметно при использовании Nested Loop, когда стоимость запроса сократилась аж в 261 раз.
