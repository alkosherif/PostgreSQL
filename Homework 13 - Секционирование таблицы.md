# Секционирование таблицы

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

Создадим общедоступную папку и перейдём в неё:
```
cd /home
sudo mkdir distr
sudo chmod 777 distr
cd distr
```

Скачаем архив с тестовой базой:
```
wget https://edu.postgrespro.ru/demo-medium.zip
```

В консоль выведется:
```
--2024-04-05 01:59:55--  https://edu.postgrespro.ru/demo-medium.zip
Resolving edu.postgrespro.ru (edu.postgrespro.ru)... 213.171.56.196
Connecting to edu.postgrespro.ru (edu.postgrespro.ru)|213.171.56.196|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 64544914 (62M) [application/zip]
Saving to: ‘demo-medium.zip’

demo-medium.zip     100%[==================>]  61,55M  9,24MB/s    in 6,3s    

2024-04-05 02:00:02 (9,70 MB/s) - ‘demo-medium.zip’ saved [64544914/64544914]
```

Выдадим всем права на архив и вызовем zcat под пользователем postgres:
```
sudo chmod 777 demo-medium.zip
sudo su postgres
zcat ./demo-medium.zip | psql
```

В консоли увидим, что происходят изменения в БД (выборочно показаны некоторые строки из большого списка):
```
SET
CREATE DATABASE
You are now connected to database "demo" as user "postgres".
SET
CREATE SCHEMA
CREATE EXTENSION
CREATE FUNCTION
CREATE TABLE
```

