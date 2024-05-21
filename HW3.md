# Домашнее задание №3
Настройка дисков для Постгреса

**Цель:**
* создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
* переносить содержимое базы данных PostgreSQL на дополнительный диск
* переносить содержимое БД PostgreSQL между виртуальными машинами

## Установка PostgreSQL 16 на виртуальную машину.
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql
```
## Проверка, что кластер запущен
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
## Создание произвольной таблицы с данными
```
postgres=# CREATE TABLE test2 (i serial, amount int);
CREATE TABLE
postgres=# INSERT INTO test2(amount) VALUES (100),(500);
INSERT 0 2
postgres=# select * from test2;
 i | amount
---+--------
 1 |    100
 2 |    500
(2 rows)
```
## Остановка postgres
```
root@otus3:~# sudo -u postgres pg_ctlcluster 16 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main
root@otus3:~# systemctl stop postgresql@16-main
```
## Создание нового диска vdb
```
root@otus3:~# sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE     SIZE MOUNTPOINT        LABEL
loop0  squashfs  63.3M /snap/core20/1822
loop1  squashfs 111.9M /snap/lxd/24322
loop2  squashfs    87M /snap/lxd/28373
loop3  squashfs  49.8M /snap/snapd/18357
loop4  squashfs  38.7M /snap/snapd/21465
loop5            63.9M /snap/core20/2318
vda                20G
├─vda1              1M
└─vda2 ext4        20G /
vdb                10G
```
## Форматирование под ext4
```
root@otus3:/mnt# mkfs.ext4 /dev/vdb
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 0cdd55d2-d462-4048-a474-d2f131b047c7
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
## Создание директории и монтирование её на диск vdb
```
root@otus3:/mnt# mkdir /mnt/vdb1
root@otus3:/mnt# mount /dev/vdb /mnt/vdb1
```
## Проверка диска после перезапуска ВМ
```
root@otus3:/mnt/vdb1# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.9M  1 loop /snap/core20/2318
loop1    7:1    0  63.3M  1 loop /snap/core20/1822
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0    87M  1 loop /snap/lxd/28373
loop4    7:4    0  49.8M  1 loop /snap/snapd/18357
loop5    7:5    0  38.7M  1 loop /snap/snapd/21465
vda    252:0    0    20G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    20G  0 part /
vdb    252:16   0    10G  0 disk /mnt/vdb1
```
## Перенос данных БД на новый диск
```
root@otus3:/mnt/vdb1# mv /var/lib/postgresql/16 /mnt/vdb1
```

## Попытка запуска postgres
```
root@otus3:/mnt/vdb1# sudo -u postgres pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
systemctl start postgresql@16-main
```
Запуск неудачный, так как используются данные из каталога /var/lib/postgresql/16/main, но они были перенесены на новый диск

## Изменение директории с данными в конфигурационном файле /etc/postgresql/16/main/postgresql.conf
Изменяем параметр
```
data_directory = '/var/lib/postgresql/16/main' # use data in another directory
```
До вида:
```
data_directory = '/mnt/vdb1/data/16/main'  
```
## Запуск postgres после изменений
```
root@otus3:/mnt/vdb1/data# systemctl start postgresql@16-main
root@otus3:/mnt/vdb1/data# systemctl status postgresql@16-main
● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Tue 2024-05-21 12:28:32 UTC; 6s ago
root@otus3:/mnt/vdb1/data# sudo -u postgres pg_lsclusters	 
Ver Cluster Port Status Owner    Data directory         Log file
16  main    5432 online postgres /mnt/vdb1/data/16/main /var/log/postgresql/postgresql-16-main.log
```
Запуск удачный, так путь до директории с данными в конфигурационном файле /etc/postgresql/16/main/postgresql.conf был изменен
## Проверка содержимового тестовой таблицы после переноса данных на новый диск
```
postgres=# select * from test2;
 i | amount
---+--------
 1 |    100
 2 |    500
(2 rows)
```