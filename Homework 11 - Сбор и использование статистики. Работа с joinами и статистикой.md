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

### Создание таблиц

Будем использовать следующие таблицы:
* Предметы (items)
* Цвет (colors)
* Форма (shapes)

```sql
create table colors (
    Id serial primary key,
    Name varchar(100) not null,
    Code varchar(10) not null,
);

create table shapes (
    Id serial primary key,
    Name varchar(100) not null
);

create table items (
    Id serial primary key,
    Description varchar (100) not null,
    ColorId int null,
    ShapeId int null,
    constraint fk_color
      foreign key(ColorId) 
        references colors(Id)
    constraint fk_shape
      foreign key(ShapeId) 
        references shapes(Id)
);
```

В консоли на каждую таблицу выведется:
```
CREATE TABLE
```

#### Добавим немного правдоподобных данных

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

-- Добавим предметы:
insert into items (Description, ColorId, ShapeId) values
('Красный круг', 1, 1),
('Чёрный квадрат', 8, 4),
('Море', 3, null),
('Тюльпан', 3, null),
('Бермудский треугольник', null, 3),
('Хоровод', null, 1),
('Синий куб', 3, 5),
('Зелёная трапеция', 2, 8)
;
```

В консоль выведется:
```
INSERT 0 14
INSERT 0 24
INSERT 0 8
```


## Выполнение ДЗ

### Реализовать прямое соединение двух или более таблиц
### Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
### Реализовать кросс соединение двух или более таблиц
### Реализовать полное соединение двух или более таблиц
### Реализовать запрос, в котором будут использованы разные типы соединений
### Сделать комментарии на каждый запрос
### К работе приложить структуру таблиц, для которых выполнялись соединения
