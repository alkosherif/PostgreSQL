# Подготовка

## Создание ВМ

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
8. В поле "Логин" указываем "otus"  
9. В поле "SSH-ключ" указываем публичный ключ:  
9.1. Заходим в PuttyGen  
9.2. Нажимаем "Generate"  
9.3. Генерируем ключи, двигая мышкой по пустой области  
9.4. Сохраняем публичный и приватный ключи в отдельную папку  
9.5. Копируем текст из поля Key (Public key for pasting into Open SSH authorized_keys file)  
9.6. Вставляем в поле "SSH-ключ"  
10. Проставляем галочку "Разрешить доступ к серийной консоли"
11. Нажимаем "Создать ВМ"
12. В открывшемся окне ждём завершения создания ВМ и сохраняем публичный IPv4 (158.160.142.17)

## Подключение к ВМ

1. Открываем Putty
1. Нажимаем Add Key
1. Выбираем сохранённый .ppk и закрываем окно
1. Нажимаем правой кнопкой на иконку Putty в трее и выбираем New Session
1. В поле "Host Name" вбиваем созранённый ранее IPv4 (158.160.142.17)
1. Нажимаем кнопку "Open"
1. В появившемся окне нажимаем "Accept"
1. В появившейся консоли в ответ на вопрос "login as" пишем ранее указанный в ВМ логин otus

Теперь можно выполнять команды.

# Работа с контейнерами

### Установка Docker
Выполняем команду:
```
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
```

В консоль выведется:

```
# Executing docker install script, commit: e5543d473431b782227f8908005543bb4389b8de  
+ sh -c apt-get update -qq >/dev/null  
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null  
+ sh -c install -m 0755 -d /etc/apt/keyrings  
+ sh -c curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg  
+ sh -c chmod a+r /etc/apt/keyrings/docker.gpg  
+ sh -c echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker.list  
+ sh -c apt-get update -qq >/dev/null  
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-ce-rootless-extras docker-buildx-plugin >/dev/null  
+ sh -c docker version  
Client: Docker Engine - Community  
 Version:           24.0.7  
 API version:       1.43  
 Go version:        go1.20.10  
 Git commit:        afdd53b  
 Built:             Thu Oct 26 09:08:01 2023  
 OS/Arch:           linux/amd64  
 Context:           default  

Server: Docker Engine - Community  
 Engine:  
  Version:          24.0.7  
  API version:      1.43 (minimum version 1.12)  
  Go version:       go1.20.10  
  Git commit:       311b9ff  
  Built:            Thu Oct 26 09:08:01 2023  
  OS/Arch:          linux/amd64  
  Experimental:     false  
 containerd:  
  Version:          1.6.26  
  GitCommit:        3dd1e886e55dd695541fdcd67420c2888645a495  
 runc:  
  Version:          1.1.10  
  GitCommit:        v1.1.10-0-g18a0cb0  
 docker-init:  
  Version:          0.19.0  
  GitCommit:        de40ad0  

================================================================================  

To run Docker as a non-privileged user, consider setting up the  
Docker daemon in rootless mode for your user:  

    dockerd-rootless-setuptool.sh install  

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.  


To run the Docker daemon as a fully privileged service, but granting non-root  
users access, refer to https://docs.docker.com/go/daemon-access/  

WARNING: Access to the remote API on a privileged Docker daemon is equivalent  
         to root access on the host. Refer to the 'Docker daemon attack surface'  
         documentation for details: https://docs.docker.com/go/attack-surface/  

================================================================================  
```

### Создание docker-сети для взаимодействия с контейнером
Выполняем команду:
```
sudo docker network create pg-net
```

В консоли выведется что-то вроде:
```
94a3308146ee5845d517a4cd37aab6126cf7fc209c7b2dda6e7445b9387fbcb0
```

### Подключение созданной сети к контейнеру *сервера* Postgres
Выполняем команду:
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```

Здесь:
* "-name pg-server" задаёт название контейнера
* "--network pg-net" указывает, что нужно использовать ранее созданную сеть "pg-net"
* "POSTGRES_PASSWORD=postgres" задаёт пароль "postgres" для подключения
* "-p 5432:5432" задаёт порты хоста и контейнера соответственно
* "-v /var/lib/postgres:/var/lib/postgresql/data" виртуализирует хостовую директорию ВМ, которая будет использоваться для хранения данных контейнера
* "postgres:15" задаёт версию postgres

В консоли выведется:
```
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
af107e978371: Pull complete
e058610431c2: Pull complete
6e27a1cf8d43: Pull complete
b4c9c895745a: Pull complete
e64f0851a8bf: Pull complete
3659855e1246: Pull complete
f400569ac77e: Pull complete
4c2b61b418d2: Pull complete
f685b32afa11: Pull complete
38e465e593f5: Pull complete
9753d74fdbc5: Pull complete
52a0696ec2c6: Pull complete
9d68039f8fea: Pull complete
1ef9664e3afd: Pull complete
Digest: sha256:7415b9cc4d7f50194a4b7b2314ff2be0e6bcba82ebfd478991c2d846d2b8af78
Status: Downloaded newer image for postgres:15
023316193f6fb55295e3dfd05ef444eae0911eb7e5d323dd1521968e535a0c45
```

### Запуск контейнера с *клиентом* в сети pg-net с БД

Выполняем команду:
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```

Вводим пароль "postgres", указанный ранее.

**Теперь можно вводить команды PostgreSQL.**

## Работа с PostgreSQL

### Создание и заполнение БД
Выполняем команду:
```SQL
create database otus;
```

В консоли выведется:
```
CREATE DATABASE
```

Добавим таблицу:

```SQL
create table testData (id int, data varchar(100));
```

В консоли выведется:
```
CREATE TABLE
```

Добавим две строки в созданную таблицу:
```SQL
insert into testData (data) values ('test1'),('test2');
```

В консоли выведется:
```
INSERT 0 2
```

Проверим, что данные добавились:
```SQL
select * from testData;
```

В консоли видим таблицу:
```
 id | data
----+-------
    | test1
    | test2
(2 rows)
```

Вводим команду \l и в выведенной таблице ищем название созданной БД, чтобы убедиться, что она создалась.
Выходим командой \q.

### Проверка, что выполнялось подключение через отдельный контейнер:

Выполняем команду:
```
sudo docker ps -a
```

В консоли видим:
```
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
023316193f6f   postgres:15   "docker-entrypoint.s…"   19 minutes ago   Up 19 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```

## Подключение с домашнего компьютера

Попробуем подключиться к контейнеру с домашнего ПК с помощью WSL в Windows:

1. Открываем командную строку (cmd.exe)
2. Вводим команду:
   bash
   
В консоли выведется приветственная информация и появится возможность ввода команд linux:
   
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.133.1-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This message is shown once a day. To disable it please create the
/home/username/.hushlogin file.
   
3. Подключаемся к PostgreSQL используя публичный IP, записанный ранее: 

psql -p 5432 -U postgres -h 158.160.142.17 -d postgres -W

Получаем ошибку:

Command 'psql' not found, but can be installed with:
sudo apt install postgresql-client-common

4. Пробуем выполнить команду, полученную в информации об ошибке:

sudo apt install postgresql-client-common

Вводим пароль по требованию.
В консоль выведется:

Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  postgresql-client-common
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 29.6 kB of archives.
After this operation, 194 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu jammy/main amd64 postgresql-client-common all 238 [29.6 kB]
Fetched 29.6 kB in 0s (60.0 kB/s)
Selecting previously unselected package postgresql-client-common.
(Reading database ... 24208 files and directories currently installed.)
Preparing to unpack .../postgresql-client-common_238_all.deb ...
Unpacking postgresql-client-common (238) ...
Setting up postgresql-client-common (238) ...
Processing triggers for man-db (2.10.2-1) ...

5. Снова пробуем подключиться к контейнеру:

psql -p 5432 -U postgres -h 158.160.142.17 -d postgres -W

На этот раз в консоль выведется другая ошибка:

Error: You must install at least one postgresql-client-<version> package

6. Устанавливаем клиент PostgreSQL:

sudo apt-get install postgresql-client

В консоль выведется:

Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libpq5 postgresql-client-14
Suggested packages:
  postgresql-14 postgresql-doc-14
The following NEW packages will be installed:
  libpq5 postgresql-client postgresql-client-14
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 1366 kB of archives.
After this operation, 4397 kB of additional disk space will be used.
Do you want to continue? [Y/n]

7. Вводим Y и продолжаем:

Ign:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpq5 amd64 14.9-0ubuntu0.22.04.1
Ign:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 postgresql-client-14 amd64 14.9-0ubuntu0.22.04.1
Get:3 http://archive.ubuntu.com/ubuntu jammy/main amd64 postgresql-client all 14+238 [3292 B]
Err:1 http://security.ubuntu.com/ubuntu jammy-updates/main amd64 libpq5 amd64 14.9-0ubuntu0.22.04.1
  404  Not Found [IP: 185.125.190.39 80]
Err:2 http://security.ubuntu.com/ubuntu jammy-updates/main amd64 postgresql-client-14 amd64 14.9-0ubuntu0.22.04.1
  404  Not Found [IP: 185.125.190.39 80]
Fetched 3292 B in 1s (4301 B/s)
E: Failed to fetch http://security.ubuntu.com/ubuntu/pool/main/p/postgresql-14/libpq5_14.9-0ubuntu0.22.04.1_amd64.deb  404  Not Found [IP: 185.125.190.39 80]
E: Failed to fetch http://security.ubuntu.com/ubuntu/pool/main/p/postgresql-14/postgresql-client-14_14.9-0ubuntu0.22.04.1_amd64.deb  404  Not Found [IP: 185.125.190.39 80]
E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?

Как видно, установка не удалась

8. Пробуем обновить список репозиториев:
sudo apt upgrade

Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages have been kept back:
  python3-update-manager update-manager-core
The following packages will be upgraded:
  binutils binutils-common binutils-x86-64-linux-gnu curl distro-info distro-info-data irqbalance libbinutils libc-bin
  libc6 libcryptsetup12 libctf-nobfd0 libctf0 libcurl3-gnutls libcurl4 libperl5.34 libpython3.10 libpython3.10-minimal
  libpython3.10-stdlib libsqlite3-0 libssh-4 locales openssh-client perl perl-base perl-modules-5.34
  python3-cryptography python3-distro-info python3-software-properties python3.10 python3.10-minimal
  software-properties-common systemd-hwe-hwdb tar vim vim-common vim-runtime vim-tiny xxd
39 upgraded, 0 newly installed, 0 to remove and 2 not upgraded.
31 standard LTS security updates
Need to get 41.5 MB of archives.
After this operation, 20.5 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libperl5.34 amd64 5.34.0-3ubuntu1.3 [4820 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 perl amd64 5.34.0-3ubuntu1.3 [232 kB]
Get:3 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 perl-base amd64 5.34.0-3ubuntu1.3 [1762 kB]
Get:4 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 perl-modules-5.34 all 5.34.0-3ubuntu1.3 [2976 kB]
Get:5 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc6 amd64 2.35-0ubuntu3.5 [3235 kB]
Get:6 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 tar amd64 1.34+dfsg-1ubuntu0.1.22.04.2 [295 kB]
Get:7 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libc-bin amd64 2.35-0ubuntu3.5 [706 kB]
Get:8 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10 amd64 3.10.12-1~22.04.3 [1948 kB]
Get:9 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3.10 amd64 3.10.12-1~22.04.3 [508 kB]
Get:10 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10-stdlib amd64 3.10.12-1~22.04.3 [1848 kB]
Get:11 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3.10-minimal amd64 3.10.12-1~22.04.3 [2242 kB]
Get:12 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpython3.10-minimal amd64 3.10.12-1~22.04.3 [812 kB]
Get:13 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libsqlite3-0 amd64 3.37.2-2ubuntu0.3 [641 kB]
Get:14 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 irqbalance amd64 1.8.0-1ubuntu0.2 [47.2 kB]
Get:15 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 distro-info-data all 0.52ubuntu0.6 [5160 B]
Get:16 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 distro-info amd64 1.1ubuntu0.2 [18.7 kB]
Get:17 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcryptsetup12 amd64 2:2.4.3-1ubuntu1.2 [211 kB]
Get:18 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 locales all 2.35-0ubuntu3.5 [4245 kB]
Get:19 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim amd64 2:8.2.3995-1ubuntu2.15 [1735 kB]
Get:20 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-tiny amd64 2:8.2.3995-1ubuntu2.15 [710 kB]
Get:21 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-runtime all 2:8.2.3995-1ubuntu2.15 [6835 kB]
Get:22 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 xxd amd64 2:8.2.3995-1ubuntu2.15 [55.2 kB]
Get:23 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 vim-common all 2:8.2.3995-1ubuntu2.15 [81.5 kB]
Get:24 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 openssh-client amd64 1:8.9p1-3ubuntu0.6 [906 kB]
Get:25 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-distro-info all 1.1ubuntu0.2 [6554 B]
Get:26 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libctf0 amd64 2.38-4ubuntu2.4 [103 kB]
Get:27 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libctf-nobfd0 amd64 2.38-4ubuntu2.4 [108 kB]
Get:28 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 binutils-x86-64-linux-gnu amd64 2.38-4ubuntu2.4 [2327 kB]
Get:29 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libbinutils amd64 2.38-4ubuntu2.4 [662 kB]
Get:30 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 binutils amd64 2.38-4ubuntu2.4 [3194 B]
Get:31 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 binutils-common amd64 2.38-4ubuntu2.4 [222 kB]
Get:32 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libssh-4 amd64 0.9.6-2ubuntu0.22.04.2 [186 kB]
Get:33 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 curl amd64 7.81.0-1ubuntu1.15 [194 kB]
Get:34 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcurl4 amd64 7.81.0-1ubuntu1.15 [289 kB]
Get:35 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libcurl3-gnutls amd64 7.81.0-1ubuntu1.15 [284 kB]
Get:36 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-cryptography amd64 3.4.8-1ubuntu2.1 [236 kB]
Get:37 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 software-properties-common all 0.99.22.9 [14.1 kB]
Get:38 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 python3-software-properties all 0.99.22.9 [28.8 kB]
Get:39 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 systemd-hwe-hwdb all 249.11.4 [2978 B]
Fetched 41.5 MB in 6s (6496 kB/s)
Extracting templates from packages: 100%
Preconfiguring packages ...
(Reading database ... 24245 files and directories currently installed.)
Preparing to unpack .../libperl5.34_5.34.0-3ubuntu1.3_amd64.deb ...
Unpacking libperl5.34:amd64 (5.34.0-3ubuntu1.3) over (5.34.0-3ubuntu1.2) ...
Preparing to unpack .../perl_5.34.0-3ubuntu1.3_amd64.deb ...
Unpacking perl (5.34.0-3ubuntu1.3) over (5.34.0-3ubuntu1.2) ...
Preparing to unpack .../perl-base_5.34.0-3ubuntu1.3_amd64.deb ...
Unpacking perl-base (5.34.0-3ubuntu1.3) over (5.34.0-3ubuntu1.2) ...
Setting up perl-base (5.34.0-3ubuntu1.3) ...
(Reading database ... 24245 files and directories currently installed.)
Preparing to unpack .../perl-modules-5.34_5.34.0-3ubuntu1.3_all.deb ...
Unpacking perl-modules-5.34 (5.34.0-3ubuntu1.3) over (5.34.0-3ubuntu1.2) ...
Preparing to unpack .../libc6_2.35-0ubuntu3.5_amd64.deb ...
Unpacking libc6:amd64 (2.35-0ubuntu3.5) over (2.35-0ubuntu3.4) ...
Setting up libc6:amd64 (2.35-0ubuntu3.5) ...
(Reading database ... 24245 files and directories currently installed.)
Preparing to unpack .../tar_1.34+dfsg-1ubuntu0.1.22.04.2_amd64.deb ...
Unpacking tar (1.34+dfsg-1ubuntu0.1.22.04.2) over (1.34+dfsg-1ubuntu0.1.22.04.1) ...
Setting up tar (1.34+dfsg-1ubuntu0.1.22.04.2) ...
(Reading database ... 24245 files and directories currently installed.)
Preparing to unpack .../libc-bin_2.35-0ubuntu3.5_amd64.deb ...
Unpacking libc-bin (2.35-0ubuntu3.5) over (2.35-0ubuntu3.4) ...
Setting up libc-bin (2.35-0ubuntu3.5) ...
/sbin/ldconfig.real: /usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link

(Reading database ... 24245 files and directories currently installed.)
Preparing to unpack .../00-libpython3.10_3.10.12-1~22.04.3_amd64.deb ...
Unpacking libpython3.10:amd64 (3.10.12-1~22.04.3) over (3.10.12-1~22.04.2) ...
Preparing to unpack .../01-python3.10_3.10.12-1~22.04.3_amd64.deb ...
Unpacking python3.10 (3.10.12-1~22.04.3) over (3.10.12-1~22.04.2) ...
Preparing to unpack .../02-libpython3.10-stdlib_3.10.12-1~22.04.3_amd64.deb ...
Unpacking libpython3.10-stdlib:amd64 (3.10.12-1~22.04.3) over (3.10.12-1~22.04.2) ...
Preparing to unpack .../03-python3.10-minimal_3.10.12-1~22.04.3_amd64.deb ...
Unpacking python3.10-minimal (3.10.12-1~22.04.3) over (3.10.12-1~22.04.2) ...
Preparing to unpack .../04-libpython3.10-minimal_3.10.12-1~22.04.3_amd64.deb ...
Unpacking libpython3.10-minimal:amd64 (3.10.12-1~22.04.3) over (3.10.12-1~22.04.2) ...
Preparing to unpack .../05-libsqlite3-0_3.37.2-2ubuntu0.3_amd64.deb ...
Unpacking libsqlite3-0:amd64 (3.37.2-2ubuntu0.3) over (3.37.2-2ubuntu0.1) ...
Preparing to unpack .../06-irqbalance_1.8.0-1ubuntu0.2_amd64.deb ...
Unpacking irqbalance (1.8.0-1ubuntu0.2) over (1.8.0-1ubuntu0.1) ...
Preparing to unpack .../07-distro-info-data_0.52ubuntu0.6_all.deb ...
Unpacking distro-info-data (0.52ubuntu0.6) over (0.52ubuntu0.5) ...
Preparing to unpack .../08-distro-info_1.1ubuntu0.2_amd64.deb ...
Unpacking distro-info (1.1ubuntu0.2) over (1.1ubuntu0.1) ...
Preparing to unpack .../09-libcryptsetup12_2%3a2.4.3-1ubuntu1.2_amd64.deb ...
Unpacking libcryptsetup12:amd64 (2:2.4.3-1ubuntu1.2) over (2:2.4.3-1ubuntu1.1) ...
Preparing to unpack .../10-locales_2.35-0ubuntu3.5_all.deb ...
Unpacking locales (2.35-0ubuntu3.5) over (2.35-0ubuntu3.4) ...
Preparing to unpack .../11-vim_2%3a8.2.3995-1ubuntu2.15_amd64.deb ...
Unpacking vim (2:8.2.3995-1ubuntu2.15) over (2:8.2.3995-1ubuntu2.13) ...
Preparing to unpack .../12-vim-tiny_2%3a8.2.3995-1ubuntu2.15_amd64.deb ...
Unpacking vim-tiny (2:8.2.3995-1ubuntu2.15) over (2:8.2.3995-1ubuntu2.13) ...
Preparing to unpack .../13-vim-runtime_2%3a8.2.3995-1ubuntu2.15_all.deb ...
Unpacking vim-runtime (2:8.2.3995-1ubuntu2.15) over (2:8.2.3995-1ubuntu2.13) ...
Preparing to unpack .../14-xxd_2%3a8.2.3995-1ubuntu2.15_amd64.deb ...
Unpacking xxd (2:8.2.3995-1ubuntu2.15) over (2:8.2.3995-1ubuntu2.13) ...
Preparing to unpack .../15-vim-common_2%3a8.2.3995-1ubuntu2.15_all.deb ...
Unpacking vim-common (2:8.2.3995-1ubuntu2.15) over (2:8.2.3995-1ubuntu2.13) ...
Preparing to unpack .../16-openssh-client_1%3a8.9p1-3ubuntu0.6_amd64.deb ...
Unpacking openssh-client (1:8.9p1-3ubuntu0.6) over (1:8.9p1-3ubuntu0.4) ...
Preparing to unpack .../17-python3-distro-info_1.1ubuntu0.2_all.deb ...
Unpacking python3-distro-info (1.1ubuntu0.2) over (1.1ubuntu0.1) ...
Preparing to unpack .../18-libctf0_2.38-4ubuntu2.4_amd64.deb ...
Unpacking libctf0:amd64 (2.38-4ubuntu2.4) over (2.38-4ubuntu2.3) ...
Preparing to unpack .../19-libctf-nobfd0_2.38-4ubuntu2.4_amd64.deb ...
Unpacking libctf-nobfd0:amd64 (2.38-4ubuntu2.4) over (2.38-4ubuntu2.3) ...
Preparing to unpack .../20-binutils-x86-64-linux-gnu_2.38-4ubuntu2.4_amd64.deb ...
Unpacking binutils-x86-64-linux-gnu (2.38-4ubuntu2.4) over (2.38-4ubuntu2.3) ...
Preparing to unpack .../21-libbinutils_2.38-4ubuntu2.4_amd64.deb ...
Unpacking libbinutils:amd64 (2.38-4ubuntu2.4) over (2.38-4ubuntu2.3) ...
Preparing to unpack .../22-binutils_2.38-4ubuntu2.4_amd64.deb ...
Unpacking binutils (2.38-4ubuntu2.4) over (2.38-4ubuntu2.3) ...
Preparing to unpack .../23-binutils-common_2.38-4ubuntu2.4_amd64.deb ...
Unpacking binutils-common:amd64 (2.38-4ubuntu2.4) over (2.38-4ubuntu2.3) ...
Preparing to unpack .../24-libssh-4_0.9.6-2ubuntu0.22.04.2_amd64.deb ...
Unpacking libssh-4:amd64 (0.9.6-2ubuntu0.22.04.2) over (0.9.6-2ubuntu0.22.04.1) ...
Preparing to unpack .../25-curl_7.81.0-1ubuntu1.15_amd64.deb ...
Unpacking curl (7.81.0-1ubuntu1.15) over (7.81.0-1ubuntu1.14) ...
Preparing to unpack .../26-libcurl4_7.81.0-1ubuntu1.15_amd64.deb ...
Unpacking libcurl4:amd64 (7.81.0-1ubuntu1.15) over (7.81.0-1ubuntu1.14) ...
Preparing to unpack .../27-libcurl3-gnutls_7.81.0-1ubuntu1.15_amd64.deb ...
Unpacking libcurl3-gnutls:amd64 (7.81.0-1ubuntu1.15) over (7.81.0-1ubuntu1.14) ...
Preparing to unpack .../28-python3-cryptography_3.4.8-1ubuntu2.1_amd64.deb ...
Unpacking python3-cryptography (3.4.8-1ubuntu2.1) over (3.4.8-1ubuntu2) ...
Preparing to unpack .../29-software-properties-common_0.99.22.9_all.deb ...
Unpacking software-properties-common (0.99.22.9) over (0.99.22.8) ...
Preparing to unpack .../30-python3-software-properties_0.99.22.9_all.deb ...
Unpacking python3-software-properties (0.99.22.9) over (0.99.22.8) ...
Preparing to unpack .../31-systemd-hwe-hwdb_249.11.4_all.deb ...
Unpacking systemd-hwe-hwdb (249.11.4) over (249.11.3) ...
Setting up irqbalance (1.8.0-1ubuntu0.2) ...
Setting up distro-info-data (0.52ubuntu0.6) ...
Setting up openssh-client (1:8.9p1-3ubuntu0.6) ...
Setting up libsqlite3-0:amd64 (3.37.2-2ubuntu0.3) ...
Setting up binutils-common:amd64 (2.38-4ubuntu2.4) ...
Setting up libctf-nobfd0:amd64 (2.38-4ubuntu2.4) ...
Setting up perl-modules-5.34 (5.34.0-3ubuntu1.3) ...
Setting up locales (2.35-0ubuntu3.5) ...
Generating locales (this might take a while)...
Generation complete.
Setting up xxd (2:8.2.3995-1ubuntu2.15) ...
Setting up vim-common (2:8.2.3995-1ubuntu2.15) ...
Setting up python3-software-properties (0.99.22.9) ...
Setting up python3-cryptography (3.4.8-1ubuntu2.1) ...
Setting up libpython3.10-minimal:amd64 (3.10.12-1~22.04.3) ...
Setting up libssh-4:amd64 (0.9.6-2ubuntu0.22.04.2) ...
Setting up systemd-hwe-hwdb (249.11.4) ...
Setting up libcurl4:amd64 (7.81.0-1ubuntu1.15) ...
Setting up libcryptsetup12:amd64 (2:2.4.3-1ubuntu1.2) ...
Setting up curl (7.81.0-1ubuntu1.15) ...
Setting up libbinutils:amd64 (2.38-4ubuntu2.4) ...
Setting up vim-runtime (2:8.2.3995-1ubuntu2.15) ...
Setting up python3-distro-info (1.1ubuntu0.2) ...
Setting up libctf0:amd64 (2.38-4ubuntu2.4) ...
Setting up distro-info (1.1ubuntu0.2) ...
Setting up libperl5.34:amd64 (5.34.0-3ubuntu1.3) ...
Setting up libcurl3-gnutls:amd64 (7.81.0-1ubuntu1.15) ...
Setting up vim-tiny (2:8.2.3995-1ubuntu2.15) ...
Setting up python3.10-minimal (3.10.12-1~22.04.3) ...
Setting up software-properties-common (0.99.22.9) ...
Setting up libpython3.10-stdlib:amd64 (3.10.12-1~22.04.3) ...
Setting up perl (5.34.0-3ubuntu1.3) ...
Setting up binutils-x86-64-linux-gnu (2.38-4ubuntu2.4) ...
Setting up libpython3.10:amd64 (3.10.12-1~22.04.3) ...
Setting up vim (2:8.2.3995-1ubuntu2.15) ...
Setting up python3.10 (3.10.12-1~22.04.3) ...
Setting up binutils (2.38-4ubuntu2.4) ...
Processing triggers for dbus (1.12.20-2ubuntu4.1) ...
Processing triggers for udev (249.11-0ubuntu3.11) ...
Processing triggers for libc-bin (2.35-0ubuntu3.5) ...
/sbin/ldconfig.real: /usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link

Processing triggers for man-db (2.10.2-1) ...

9. Снова пробуем установить клиент PostgreSQL:

sudo apt-get install postgresql-client

Теперь клиент 14й версии будет успешно установлен:

Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libpq5 postgresql-client-14
Suggested packages:
  postgresql-14 postgresql-doc-14
The following NEW packages will be installed:
  libpq5 postgresql-client postgresql-client-14
0 upgraded, 3 newly installed, 0 to remove and 2 not upgraded.
Need to get 1366 kB/1369 kB of archives.
After this operation, 4409 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 libpq5 amd64 14.10-0ubuntu0.22.04.1 [143 kB]
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 postgresql-client-14 amd64 14.10-0ubuntu0.22.04.1 [1223 kB]
Fetched 1366 kB in 1s (955 kB/s)
Selecting previously unselected package libpq5:amd64.
(Reading database ... 24245 files and directories currently installed.)
Preparing to unpack .../libpq5_14.10-0ubuntu0.22.04.1_amd64.deb ...
Unpacking libpq5:amd64 (14.10-0ubuntu0.22.04.1) ...
Selecting previously unselected package postgresql-client-14.
Preparing to unpack .../postgresql-client-14_14.10-0ubuntu0.22.04.1_amd64.deb ...
Unpacking postgresql-client-14 (14.10-0ubuntu0.22.04.1) ...
Selecting previously unselected package postgresql-client.
Preparing to unpack .../postgresql-client_14+238_all.deb ...
Unpacking postgresql-client (14+238) ...
Setting up libpq5:amd64 (14.10-0ubuntu0.22.04.1) ...
Setting up postgresql-client-14 (14.10-0ubuntu0.22.04.1) ...
update-alternatives: using /usr/share/postgresql/14/man/man1/psql.1.gz to provide /usr/share/man/man1/psql.1.gz (psql.1.gz) in auto mode
Setting up postgresql-client (14+238) ...
Processing triggers for libc-bin (2.35-0ubuntu3.5) ...
/sbin/ldconfig.real: /usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link

10. Пробуем снова:

psql -p 5432 -U postgres -h 158.160.142.17 -d postgres -W

Вводим пароль и теперь можно вводить команды:

psql (14.10 (Ubuntu 14.10-0ubuntu0.22.04.1), server 15.5 (Debian 15.5-1.pgdg120+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
Type "help" for help.

postgres=#

11. Посмотрим данные в таблице:

postgres=# select * from testData;
 id | data
----+-------
    | test1
    | test2
(2 rows)

12. Переходим в putty и выполняем команды для остановки и удаления контейнера:

12.1: Получим идентификатор контейнера:

docker ps

В консоль выведется:

CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                                                                         PORTS                                       NAMES
cc81fd1086ed   postgres:15   "docker-entrypoint.s…"   48 minutes ago   Up 48 min                                                               utes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

12.2 Остановим контейнер:

docker stop cc81fd1086ed

В консоль выведется:
cc81fd1086ed

12.3 Удалим контейнер:
docker rm cc81fd1086ed

В консоль выведется:
cc81fd1086ed

13. Создим контейнер с сервером заново
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15

В консоль выведется:
44d4d00f35d4a1f85f3bb946727d47a607ecb09ec75ba40d95f7101a91132d2d

14. Подключаемся снова к PostgreSQL из контейнера с клиентом:
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres: [вводим пароль]
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=# 

15. Проверяем, что даннные остались на месте:
postgres=# select * from testData;
 id | data
----+-------
    | test1
    | test2
(2 rows)


В результате видим, что не смотря на удаление контейнера данные в БД не были удалены. При повторном создании контейнера с теми же параметрами в базе будут все данные, имевшиеся на момент удаления предыдущего контейнера.

