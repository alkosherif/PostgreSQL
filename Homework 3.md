# Физический уровень PostgreSQL. Установка и настройка PostgreSQL
### Создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере

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
7.2 Выбрать 4 процессора  
8. Нажать кнопку "Готово"  
9. Подождать, пока установится ОС  
10. Пропустить все вопросы, связанные с настройкой, после запуска ОС  
11. Терминал при этом не открывается, поэтому нужно зайти в Settings > Region > Language > Login Screen > явным образом выбрать English (United States) без ISO-8859-1 и перезагрузить ОС (см. https://ru.stackoverflow.com/questions/1476869/%D0%9D%D0%B5-%D0%BC%D0%BE%D0%B3%D1%83-%D0%BE%D1%82%D0%BA%D1%80%D1%8B%D1%82%D1%8C-%D1%82%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B0%D0%BB-%D0%B2-ubuntu)  
12. Админских прав у пользователя не будет, поэтому нужно (см. https://youtu.be/jZGHtuxpaP4?si=X6mA74Qdt00wUaw6):  
12.1. Ввести команду su  
12.2. Ввести пароль текущего пользователя  
12.3. Ввести команду usermod -aG sudo vboxuser  
12.4. Перезагрузить ОС, например, командой reboot  
13. Даже если настроить в VirtualBox двунаправленный буфер обмена, он работать не будет. Чтобы это исправить, нужно:  
13.1. Скачать образ https://download.virtualbox.org/virtualbox/6.1.2/VBoxGuestAdditions_6.1.2.iso  
13.2. Смонтировать образ, например, открыв его в папке, с помощью двойного клика  
13.3. Открыть терминал в папке смонтированного образа  
13.4. Выполнить команду sudo sh VBoxLinuxAdditions.run  
13.5. Перезагрузить ОС  
13.6. После этого на экране будут постоянно появляться странные артефакты, но по крайней мере заработает общий буфер обмена. После дополнительной пары перезагрузок артефакты могут пропасть.  

### Поставьте на нее PostgreSQL 15 через sudo apt
В этом поможет следующий набор команд:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Появится длинный лог установки, можем его пропустить.

### Проверьте что кластер запущен через sudo -u postgres pg_lsclusters

Выполняем команду
```
sudo -u postgres pg_lsclusters
```

В консоль выведется:
```
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

### Зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым

Выполняем команду  
```
sudo -u postgres psql
```

Теперь можно вводить команды.  
  
Вводим:
```sql
create table test(c1 text);
insert into test values('1');
```

В консоль выведется:
```
CREATE TABLE
INSERT 0 1
```


Для выхода используем команду:
```sql
\q
```

### Остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop

Выполняем команду:
```
sudo -u postgres pg_ctlcluster 15 main stop
```
В консоль выведется:
```
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
```

### Создайте новый диск к ВМ размером 10GB

1. Завершаем работу в ОС Ubuntu  
2. Выбираем виртуальную машину  
3. Нажимаем кнопку "Настроить"  
4. В появившемся окне заходим в раздел "Носители"  
5. Выбираем "Контроллер: SATA"  
6. Нажимаем кнопку + с изображением жёсткого диска  
7. В появившемся окне Выбор жёсткого диска" нажимаем кнопку "Создать"  
8. Выбранный "RadioButton VDI (VirtualBox Disk Image)" оставляем без изменений, нажимаем "Далее"  
9. В диалоге "Укажите формат хранения" нажимаем кнопку "Далее"  
10. В диалоге "Укажите имя и размер файла" указываем 10 ГБ и нажимаем "Готово"  

### Добавьте свеже-созданный диск к виртуальной машине

1. Если все окна закрыты - повторяем первые 6 шагов предыдущего пункта  
2. В окне "Выбор жёсткого диска" нужно сделать двойной клик по новому диску  

### Проинициализируйте диск согласно инструкции и подмонтировать файловую систему

Выполняем команду:  
```
sudo apt install parted
```

Вводим пароль.  
В консоли выведется:  
```
[sudo] password for vboxuser: 
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
parted is already the newest version (3.4-2build1).
parted set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 6 not upgraded.
```

Выведем список доступных дисков:  
```
sudo parted -l | grep Error
```
Получаем ошибку:  
```
Error: /dev/sdb: unrecognised disk label
```

Получаем информацию о дисках командой:  
```
lsblk
```
В консоль выведется:  
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0  74,2M  1 loop /snap/core22/1122
loop2    7:2    0 266,6M  1 loop /snap/firefox/3836
loop3    7:3    0  91,7M  1 loop /snap/gtk-common-themes/1535
loop4    7:4    0   497M  1 loop /snap/gnome-42-2204/141
loop5    7:5    0  12,3M  1 loop /snap/snap-store/959
loop6    7:6    0  40,4M  1 loop /snap/snapd/20671
loop7    7:7    0   452K  1 loop /snap/snapd-desktop-integration/83
sda      8:0    0    25G  0 disk 
├─sda1   8:1    0     1M  0 part 
├─sda2   8:2    0   513M  0 part /boot/efi
└─sda3   8:3    0  24,5G  0 part /var/snap/firefox/common/host-hunspell
                                 /
sdb      8:16   0   10G  0 disk 
sr0     11:0    1  1024M  0 rom  
```

Создаём таблицу разделов gpt:
```
sudo parted /dev/sdb mklabel gpt
```
В консоль выведется:  
```
Information: You may need to update /etc/fstab.
```

Создаём раздел командой:
```
sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%
```
В консоль выведется:  
```
Information: You may need to update /etc/fstab.
```

Снова получаем информацию о дисках:  
```
lsblk
```
Видим:  
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0     4K  1 loop /snap/bare/5
loop1    7:1    0  74,2M  1 loop /snap/core22/1122
loop2    7:2    0 266,6M  1 loop /snap/firefox/3836
loop3    7:3    0  91,7M  1 loop /snap/gtk-common-themes/1535
loop4    7:4    0   497M  1 loop /snap/gnome-42-2204/141
loop5    7:5    0  12,3M  1 loop /snap/snap-store/959
loop6    7:6    0  40,4M  1 loop /snap/snapd/20671
loop7    7:7    0   452K  1 loop /snap/snapd-desktop-integration/83
sda      8:0    0    25G  0 disk 
├─sda1   8:1    0     1M  0 part 
├─sda2   8:2    0   513M  0 part /boot/efi
└─sda3   8:3    0  24,5G  0 part /var/snap/firefox/common/host-hunspell
                                 /
sdb      8:16   0   10G  0 disk 
└─sdb1   8:17   0   10G  0 part 
sr0     11:0    1  1024M  0 rom  
```

Создаём файловую систему в разделе жёсткого диска:  
```
sudo mkfs.ext4 -L datapartition /dev/sdb1
```

В консоль выведется:  
```
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 548864 4k blocks and 137360 inodes
Filesystem UUID: b38458a6-41e6-45ce-95f0-d7c4a4e6242d
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
```

Присваиваем метки файловой системе:  
```
sudo e2label /dev/sdb1 attached
```

Смотрим информацию о дисках, добавляя информацию о файловой системе настройкой --fs:  
```
sudo lsblk --fs
```

В консоли видим:  
```
NAME FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0
     squash 4.0                                                    0   100% /snap/bare/5
loop1
     squash 4.0                                                    0   100% /snap/core22/1122
loop2
     squash 4.0                                                    0   100% /snap/firefox/3836
loop3
     squash 4.0                                                    0   100% /snap/gtk-common-themes/1535
loop4
     squash 4.0                                                    0   100% /snap/gnome-42-2204/141
loop5
     squash 4.0                                                    0   100% /snap/snap-store/959
loop6
     squash 4.0                                                    0   100% /snap/snapd/20671
loop7
     squash 4.0                                                    0   100% /snap/snapd-desktop-integration/83
sda                                                                         
├─sda1
│                                                                           
├─sda2
│    vfat   FAT32       0BB2-2007                             505,9M     1% /boot/efi
└─sda3
     ext4   1.0         8eeeabec-ec7b-4b57-a8e7-7f964ebd8149   11,3G    48% /var/snap/firefox/common/host-hunspell
                                                                            /
sdb                                                                         
└─sdb1
     ext4   1.0   attached
                        b38458a6-41e6-45ce-95f0-d7c4a4e6242d                
sr0                        
```

Монтируем путь для файлов на диске:  
```
sudo mkdir -p /mnt/data
sudo mount -o defaults /dev/sdb1 /mnt/data
df -h -x tmpfs
```

В консоль выведется:  
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        24G   12G   12G  51% /
/dev/sda2       512M  6,1M  506M   2% /boot/efi
/dev/sdb1       10G   24K  9,9G   1% /mnt/data
```

Проверяем, что можем создать текстовый файл:  
```
echo "success" | sudo tee /mnt/data/test_file
```
Выведется:
```
success
```

Но нам нужно проверить именно содержимое файла, поэтому выполняем команду:  
```
cat /mnt/data/test_file
```

И видим:  
```
success
```

Теперь добавляем в файл fstab строку "LABEL=attached /mnt/data ext4 defaults 0 2" чтобы новый диск не исчез после перезагрузки:
```
sudo nano /etc/fstab
```

Откроется файл, в нём последняя строка добавлена нами:
```
# /etc/fstab: static file system information.  
#  
# Use 'blkid' to print the universally unique identifier for a  
# device; this may be used with UUID= as a more robust way to name devices  
# that works even if disks are added and removed. See fstab(5).  
#  
# <file system> <mount point>   <type>  <options>       <dump>  <pass>  
# / was on /dev/sda3 during installation  
UUID=8eeeabec-ec7b-4b57-a8e7-7f964ebd8149 /               ext4    errors=remount-ro 0       1  
# /boot/efi was on /dev/sda2 during installation  
UUID=0BB2-2007  /boot/efi       vfat    umask=0077      0       1  
/swapfile                                 none            swap    sw              0       0  
LABEL=attached /mnt/data ext4 defaults 0 2  
```

### Перезагрузите инстанс и убедитесь, что диск остается примонтированным

Проверяем, что вывод команд не изменился:  
```
sudo lsblk --fs
echo "success" | sudo tee /mnt/data/test_file
cat /mnt/data/test_file
```
Вывод будет соответствовать приведённому ранее.

### Сделайте пользователя postgres владельцем /mnt/data

Для этого потребуется выполнить команду:  
```
sudo chown -R postgres:postgres /mnt/data/
```

### Перенесите содержимое /var/lib/postgres/15 в /mnt/data

Для этого потребуется выполнить команду:  
```
sudo mv /var/lib/postgresql/15 /mnt/data
```

### Попытайтесь запустить кластер

Для этого потребуется выполнить команду:  
```
sudo -u postgres pg_ctlcluster 15 main start
```

В консоль выведется:  
```
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```

### Напишите получилось или нет и почему

Так как файлы были перемещены в /mnt/data, ошибка была ожидаема.  

### Задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его. Напишите что и почему поменяли

Открываем конфигурацию PostgreSQL:  
```
sudo nano /etc/postgresql/15/main/postgresql.conf
```

В файле находим раздел "FILE LOCATIONS", в нём меняем data_directory на '/mnt/data/15/main/'.  
Выходим с сохранением изменений.  

### Попытайтесь запустить кластер
Для этого потребуется выполнить команду:  
```
sudo -u postgres pg_ctlcluster 15 main start
```

В консоль выведется:
```
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Removed stale pid file.
```

### Напишите получилось или нет и почему

Исходя из сообщения, лучше запускать кластер командой sudo systemctl start postgresql@15-main, но т.к. это Warning, а не ошибка, скорее всего он работает (что в дальнейшем подтвердится при попытке выполнять команды SQL).

### Зайдите через через psql и проверьте содержимое ранее созданной таблицы

Для этого потребуется выполнить команду:  
```
sudo -u postgres psql
```

В консоль выведется:  
```
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.
```
Теперь можно выполнять команды SQL.  

```sql
select * from test;
```

В консоль выведется то, что ожидалось:  
```
 c1 
----
 1
(1 row)
```
