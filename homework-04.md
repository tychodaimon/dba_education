## Углубленный анализ производительности. Профилирование. Мониторинг. Оптимизация 
### Cоздал виртуальную машину dba4 c Ubuntu 22.04 LTS в ЯО 
#### Установил на нее PostgreSQL 15 и инструменты профилирования и мониторинга
```
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15 htop iotop atop unzip pgtop lynx iftop pgbadger
```

#### Прогнал pgbench на дефолтных настройках 
```
dba@dba4:~$ sudo su -
root@dba4:~# su - postgres
postgres@dba4:~$ pgbench -i -s 100
dropping old tables...
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 35.27 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 52.53 s (drop tables 0.16 s, create tables 0.01 s, client-side generate 35.39 s, vacuum 0.18 s, primary keys 16.79 s).

postgres@dba4:~$ pgbench -t 5000 -c 4 -j 4
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 4
number of threads: 4
maximum number of tries: 1
number of transactions per client: 5000
number of transactions actually processed: 20000/20000
number of failed transactions: 0 (0.000%)
latency average = 4.084 ms
initial connection time = 7.066 ms
tps = 979.423967 (without initial connection time)
```

#### Применил агрессивные настройки оптимизации
```
root@dba4:~# vi /etc/postgresql/15/main/postgresql.conf

# Connectivity
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any

# Replication
wal_level = replica		# consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = off

# Checkpointing: 
checkpoint_timeout  = '15 min' 
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'


# WAL writing
wal_compression = on
wal_buffers = -1    # auto-tuned by Postgres till maximum of segment size (16MB by default)


# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries: 
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 2
parallel_leader_participation = on

# Advanced features 
enable_partitionwise_join = on 
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 100
wal_recycle = on

fsync = off
full_page_writes = off


root@dba4:~# service postgresql restart
root@dba4:~# echo never > /sys/kernel/mm/transparent_hugepage/enabled
root@dba4:~# cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]
root@dba4:~# su - postgres
postgres@dba4:~$ psql -c "ALTER SYSTEM SET synchronous_commit = off;"
ALTER SYSTEM
```

#### Прогнал pgbench на новых настройках
```
postgres@dba4:~$ pgbench -t 5000 -c 4 -j 4
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 4
number of threads: 4
maximum number of tries: 1
number of transactions per client: 5000
number of transactions actually processed: 20000/20000
number of failed transactions: 0 (0.000%)
latency average = 3.553 ms
initial connection time = 7.028 ms
tps = 1125.786995 (without initial connection time)
postgres@dba4:~$ pgbench -i -s 100
dropping old tables...
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 29.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 43.33 s (drop tables 0.07 s, create tables 0.02 s, client-side generate 29.32 s, vacuum 0.17 s, primary keys 13.76 s).

postgres@dba4:~$ pgbench -t 5000 -c 4 -j 4
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 4
number of threads: 4
maximum number of tries: 1
number of transactions per client: 5000
number of transactions actually processed: 20000/20000
number of failed transactions: 0 (0.000%)
latency average = 1.292 ms
initial connection time = 7.087 ms
tps = 3095.700645 (without initial connection time)
```
