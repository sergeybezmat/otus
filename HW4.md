# Домашнее задание №4
Тюнинг Постгреса

**Цель:**
* Развернуть инстанс Постгреса в ВМ в GCP
* Оптимизировать настройки

## Устанавливаем postgres на ВМ
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql
```
## Иницилизируем базу данных для pgbench
```
postgres@otus4:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.39 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.13 s, vacuum 0.03 s, primary keys 0.22 s).
```
## Запускаем pgbench
```
postgres@otus3:~$ pgbench
pgbench (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
number of transactions per client: 10
number of transactions actually processed: 10/10
number of failed transactions: 0 (0.000%)
latency average = 1.309 ms
initial connection time = 2.296 ms
tps = 763.825237 (without initial connection time)
```

## Подбираем опитимальные настройки в зависимости от характеристик сервера
Пользовался сайтом pgconfig.org.

Добавляем параметры в конфиг /etc/postgresql/16/main/postgresql.conf
```
# Memory Configuration
shared_buffers = 1GB
effective_cache_size = 3GB
work_mem = 10MB
maintenance_work_mem = 205MB

# Checkpoint Related Configuration
min_wal_size = 2GB
max_wal_size = 3GB
checkpoint_completion_target = 0.9
wal_buffers = -1

# Network Related Configuration
listen_addresses = '*'
max_connections = 100

# Storage Configuration
random_page_cost = 4.0
effective_io_concurrency = 2

# Worker Processes Configuration
max_worker_processes = 8
max_parallel_workers_per_gather = 2
max_parallel_workers = 2

# Logging configuration for pgbadger
logging_collector = on
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
lc_messages = 'C'

# Adjust the minimum time to collect the data
log_min_duration_statement = '10s'
log_autovacuum_min_duration = 0

# CSV Configuration
log_destination = 'csvlog'
```
## Перезапускаем кластер postgres
```
root@otus4:~# sudo pg_ctlcluster 16 main restart
```
## Проверяем состояние postgres
```
postgres@otus4:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main log/postgresql-%Y-%m-%d_%H%M%S.csv
```
## Проверяем, применились ли параметры
```
select * from pg_settings where name = 'work_mem';
"work_mem"	"10240"
```
## Повторно запускаем pgbench
```
postgres@otus4:~$ pgbench
pgbench (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
number of transactions per client: 10
number of transactions actually processed: 10/10
number of failed transactions: 0 (0.000%)
latency average = 1.200 ms
initial connection time = 2.468 ms
tps = 833.125052 (without initial connection time)
```

## Сравнение результатов
После изменения конфигурации согласно характеристикам сервера улучшились такие показатели, как:
* latency average - Средняя  задержка  выполнения  транзакции(c 1.309ms до 1.200ms);
* tps - Количество  транзакций  в  секунду (с 763.825237 до 833.125052)

Так как база данных является пустой, изменение параметров существенно не повлияли на производительность базы данных.
