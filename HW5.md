# Домашнее задание №5
Бэкапы Постгреса

**Цель:**
* Используем современные решения для бэкапов

## Устанавливаем WAL-G
```
wget https://github.com/wal-g/wal-g/releases/download/v2.0.1/wal-g-pg-ubuntu-20.04-amd64.tar.gz && tar -zxvf wal-g-pg-ubuntu-20.04-amd64.tar.gz && sudo mv wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
```
## Создаем директорию для бэкапов и назначаем права
```
root@otus5:~# mkdir /home/backups && chmod 777 /home/backups
```
## В домашней директории пользователя postgres создаем настроечный файл для wal-g
```
postgres@otus5:~$ pwd
/var/lib/postgresql
postgres@otus5:~$ vim .walg.json
postgres@otus5:~$ cat .walg.json
{
    "WALG_FILE_PREFIX": "/home/backups",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/16/main/",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}
```
## Создаем директорию для логов
```
postgres@otus6:~$ mkdir /var/lib/postgresql/log

```
## Добавляем параметры бэкапирования в postgresql.conf
```
echo "wal_level=replica" >> /etc/postgresql/16/main/postgresql.conf
echo "archive_mode=on" >> /etc/postgresql/16/main/postgresql.conf
echo "archive_command='/usr/local/bin/wal-g wal-push \"%p\" >> /var/lib/postgresql/log/archive_command.log 2>&1' " >> /etc/postgresql/16/main/postgresql.conf
echo "archive_timeout=60" >> /etc/postgresql/16/main/postgresql.conf
echo "restore_command='/usr/local/bin/wal-g wal-fetch \"%f\" \"%p\" >> /var/lib/postgresql/log/restore_command.log 2>&1' " >> /etc/postgresql/16/main/postgresql.conf

```
## Проверяем данные в тестовой таблице
```
postgres=# select * from test2;
 i | amount
---+--------
 1 |    100
 2 |    500
(2 rows)
```
## Делаем первый бэкап
```
/usr/local/bin/wal-g backup-push /var/lib/postgresql/16/main/
```
## Проверяем созданный бэкап
```
postgres@otus5:~$ /usr/local/bin/wal-g backup-list
name                          modified             wal_segment_backup_start
base_000000010000000000000003 2024-05-22T13:54:09Z 000000010000000000000003
```

## Добавляем еще несколько строк
```
postgres=# INSERT into test2(amount) values (700), (800);
INSERT 0 2
postgres=# select * from test2;
 i | amount
---+--------
 1 |    100
 2 |    500
 3 |    700
 4 |    800
(4 rows)
```
## Повторяем бэкап и проверяем
```
postgres@otus5:~$ /usr/local/bin/wal-g backup-list
name                                                     modified             wal_segment_backup_start
base_000000010000000000000003                            2024-05-22T13:54:09Z 000000010000000000000003
base_000000010000000000000005_D_000000010000000000000003 2024-05-22T13:59:54Z 000000010000000000000005

root@otus5:/home/backups/basebackups_005# ll
total 24
drwxr-xr-x 4 postgres postgres 4096 May 22 13:59 ./
drwxrwxrwx 3 root     root     4096 May 22 13:54 ../
drwxr-xr-x 3 postgres postgres 4096 May 22 13:54 base_000000010000000000000003/
-rw-rw-r-- 1 postgres postgres  370 May 22 13:54 base_000000010000000000000003_backup_stop_sentinel.json
drwxr-xr-x 3 postgres postgres 4096 May 22 13:59 base_000000010000000000000005_D_000000010000000000000003/
-rw-rw-r-- 1 postgres postgres  491 May 22 13:59 base_000000010000000000000005_D_000000010000000000000003_backup_stop_sentinel.json
```
## Останавливаем postgres
```
root@otus5:/var/lib/postgresql/16/main# service postgresql stop
```
## Удаляем директорию с базой
```
root@otus5:/home/backups/basebackups_005# rm -rf /var/lib/postgresql/16/main/*
```
## Восстанавливаем из бэкапа
```
su - postgres -c '/usr/local/bin/wal-g backup-fetch /var/lib/postgresql/16/main LATEST'
```
## Создаем файл-сигнал
```
su - postgres -c 'touch /var/lib/postgresql/16/main/recovery.signal'
```
## Запускаем базу
```
root@otus5:/var/lib/postgresql/16/main# service postgresql start
```
## Проверяем целостность данных
```
postgres=# select * from test2;
 i | amount
---+--------
 1 |    100
 2 |    500
 3 |    700
 4 |    800
(4 rows)
```
