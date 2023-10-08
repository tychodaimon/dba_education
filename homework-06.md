# Кластер Patroni on-premise 1

```
dba@dba7-etcd1:~$ sudo su -
root@dba7-etcd1:~# apt update && sudo apt upgrade -y && sudo apt install -y etcd
root@dba7-etcd1:~# reboot
dba@dba7-etcd1:~$ ps -aef | grep etcd
dba@dba7-etcd1:~$ sudo systemctl stop etcd

dba@dba7-etcd1:~$ sudo su -
root@dba7-etcd1:~# vi /etc/default/etcd

ETCD_NAME="dba7-etcd1"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://dba7-etcd1:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://dba7-etcd1:2380"
ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
ETCD_INITIAL_CLUSTER="etcd1=http://dba7-etcd1:2380,dba7-etcd2=http://dba7-etcd2:2380,dba7-etcd3=http://dba7-etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"

root@dba7-etcd1:~# sudo systemctl start etcd
root@dba7-etcd1:~# systemctl is-enabled etcd
enabled
```
#### остальные 2 etcd по аналогии
```
root@dba7-etcd1:~# etcdctl cluster-health
member 3cd779a92eaff20e is healthy: got healthy result from http://dba7-etcd1:2379
member ee060e59bf55511f is healthy: got healthy result from http://dba7-etcd2:2379
member f1fc6a1249152a0d is healthy: got healthy result from http://dba7-etcd3:2379
cluster is healthy

root@dba7-etcd2:~# sed -i 's/ETCD_INITIAL_CLUSTER_STATE="new"/ETCD_INITIAL_CLUSTER_STATE="existing"/g' /etc/default/etcd
```

#### Postgres
```
dba@dba7-pgsql1:~$ sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14

root@dba7-pgsql1:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

root@dba7-etcd1:~# ping dba7-pgsql1.ru-central1.internal
PING dba7-pgsql1.ru-central1.internal (10.128.0.37) 56(84) bytes of data.
64 bytes from dba7-pgsql1.ru-central1.internal (10.128.0.37): icmp_seq=1 ttl=61 time=1.35 ms
64 bytes from dba7-pgsql1.ru-central1.internal (10.128.0.37): icmp_seq=2 ttl=61 time=0.311 ms

root@dba7-pgsql1:~# ping dba7-etcd1.ru-central1.internal
PING dba7-etcd1.ru-central1.internal (10.128.0.32) 56(84) bytes of data.
64 bytes from dba7-etcd1.ru-central1.internal (10.128.0.32): icmp_seq=1 ttl=61 time=0.418 ms

postgres@dba7-pgsql1:~$ echo "\set PROMPT1 '%M %n@%/%R%# '" >> ~/.psqlrc

root@dba7-pgsql1:~# apt-get install -y python3 python3-pip git mc

root@dba7-pgsql1:~# pip3 install psycopg2-binary

root@dba7-pgsql1:~# systemctl stop postgresql@14-main

root@dba7-pgsql1:~# sudo -u postgres pg_dropcluster 14 main

root@dba7-pgsql1:~# pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file
```

#### Ставлю Patroni
```
root@dba7-pgsql1:~# sudo pip3 install patroni[etcd]

root@dba7-pgsql1:~# sudo ln -s /usr/local/bin/patroni /bin/patroni

root@dba7-pgsql1:~# vi /etc/systemd/system/patroni.service
root@dba7-pgsql1:~# cat /etc/systemd/system/patroni.service
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target

root@dba7-pgsql1:~# vi /etc/patroni.yml

root@dba7-pgsql1:~# cat /etc/patroni.yml
scope: patroni
name: dba7-pgsql1.ru-central1.internal
restapi:
  listen: dba7-pgsql1.ru-central1.internal:8008
  connect_address: dba7-pgsql1.ru-central1.internal:8008
etcd:
  hosts: dba7-etcd1.ru-central1.internal:2379,dba7-etcd2.ru-central1.internal:2379,dba7-etcd3.ru-central1.internal:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host replication replicator 10.0.0.0/8 md5
  - host all all 10.0.0.0/8 md5
  users:
    admin:
      password: admin_321
      options:
        - createrole
        - createdb
postgresql:
  listen: 127.0.0.1, dba7-pgsql1.ru-central1.internal:5432
  connect_address: dba7-pgsql1.ru-central1.internal:5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: zalando_321
    rewind:
      username: rewind_user
      password: rewind_password_321
  parameters:
    unix_socket_directories: '.'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

#### Запускаю
```
postgres@dba7-pgsql1:~$ patroni /etc/patroni.yml
2023-10-08 01:30:44,778 INFO: Selected new etcd server http://dba7-etcd3.ru-central1.internal:2379
2023-10-08 01:30:44,784 INFO: No PostgreSQL configuration items changed, nothing to reload.
2023-10-08 01:30:44,792 WARNING: Postgresql is not running.
2023-10-08 01:30:44,792 INFO: Lock owner: None; I am dba7-pgsql1.ru-central1.internal
2023-10-08 01:30:44,793 INFO: pg_controldata:
  pg_control version number: 1300
  Catalog version number: 202107181
  Database system identifier: 7287378821428936927
  Database cluster state: shut down
  pg_control last modified: Sun Oct  8 01:26:46 2023
  Latest checkpoint location: 0/194F478
  Latest checkpoint's REDO location: 0/194F478
  Latest checkpoint's REDO WAL file: 0000000B0000000000000001
  Latest checkpoint's TimeLineID: 11
  Latest checkpoint's PrevTimeLineID: 11
  Latest checkpoint's full_page_writes: on
  Latest checkpoint's NextXID: 0:734
  Latest checkpoint's NextOID: 13762
  Latest checkpoint's NextMultiXactId: 1
  Latest checkpoint's NextMultiOffset: 0
  Latest checkpoint's oldestXID: 727
  Latest checkpoint's oldestXID's DB: 1
  Latest checkpoint's oldestActiveXID: 0
  Latest checkpoint's oldestMultiXid: 1
  Latest checkpoint's oldestMulti's DB: 1
  Latest checkpoint's oldestCommitTsXid: 0
  Latest checkpoint's newestCommitTsXid: 0
  Time of latest checkpoint: Sun Oct  8 01:26:46 2023
  Fake LSN counter for unlogged rels: 0/3E8
  Minimum recovery ending location: 0/0
  Min recovery ending loc's timeline: 0
  Backup start location: 0/0
  Backup end location: 0/0
  End-of-backup record required: no
  wal_level setting: replica
  wal_log_hints setting: on
  max_connections setting: 100
  max_worker_processes setting: 8
  max_wal_senders setting: 10
  max_prepared_xacts setting: 0
  max_locks_per_xact setting: 64
  track_commit_timestamp setting: off
  Maximum data alignment: 8
  Database block size: 8192
  Blocks per segment of large relation: 131072
  WAL block size: 8192
  Bytes per WAL segment: 16777216
  Maximum length of identifiers: 64
  Maximum columns in an index: 32
  Maximum size of a TOAST chunk: 1996
  Size of a large-object chunk: 2048
  Date/time type storage: 64-bit integers
  Float8 argument passing: by value
  Data page checksum version: 1
  Mock authentication nonce: 5af3330af292c7b3718ab74c765512b73e3914f284063d3aca31dfaf791bf967

2023-10-08 01:30:44,798 INFO: Lock owner: None; I am dba7-pgsql1.ru-central1.internal
2023-10-08 01:30:44,802 INFO: starting as a secondary
2023-10-08 01:30:45,224 INFO: postmaster pid=5690
localhost:5432 - no response
2023-10-08 01:30:45.265 UTC [5690] LOG:  starting PostgreSQL 14.9 (Ubuntu 14.9-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2023-10-08 01:30:45.265 UTC [5690] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2023-10-08 01:30:45.282 UTC [5690] LOG:  listening on IPv4 address "10.128.0.37", port 5432
2023-10-08 01:30:45.287 UTC [5690] LOG:  listening on Unix socket "./.s.PGSQL.5432"
2023-10-08 01:30:45.300 UTC [5692] LOG:  database system was shut down at 2023-10-08 01:26:46 UTC
2023-10-08 01:30:45.300 UTC [5692] WARNING:  specified neither primary_conninfo nor restore_command
2023-10-08 01:30:45.300 UTC [5692] HINT:  The database server will regularly poll the pg_wal subdirectory to check for files placed there.
2023-10-08 01:30:45.300 UTC [5692] LOG:  entering standby mode
2023-10-08 01:30:45.320 UTC [5692] LOG:  consistent recovery state reached at 0/194F4F0
2023-10-08 01:30:45.320 UTC [5692] LOG:  invalid record length at 0/194F4F0: wanted 24, got 0
2023-10-08 01:30:45.321 UTC [5690] LOG:  database system is ready to accept read-only connections
localhost:5432 - accepting connections
localhost:5432 - accepting connections
2023-10-08 01:30:46,247 INFO: establishing a new patroni connection to the postgres cluster
2023-10-08 01:30:46,254 WARNING: Could not activate Linux watchdog device: Can't open watchdog device: [Errno 2] No such file or directory: '/dev/watchdog'
2023-10-08 01:30:46.257 UTC [5702] FATAL:  role "replicator" does not exist
2023-10-08 01:30:46,257 ERROR: Can not fetch local timeline and lsn from replication connection
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/dist-packages/patroni/postgresql/__init__.py", line 1032, in get_replica_timeline
    with self.get_replication_connection_cursor(**self.config.local_replication_address) as cur:
  File "/usr/lib/python3.10/contextlib.py", line 135, in __enter__
    return next(self.gen)
  File "/usr/local/lib/python3.10/dist-packages/patroni/postgresql/__init__.py", line 1027, in get_replication_connection_cursor
    with get_connection_cursor(**conn_kwargs) as cur:
  File "/usr/lib/python3.10/contextlib.py", line 135, in __enter__
    return next(self.gen)
  File "/usr/local/lib/python3.10/dist-packages/patroni/postgresql/connection.py", line 48, in get_connection_cursor
    conn = psycopg.connect(**kwargs)
  File "/usr/local/lib/python3.10/dist-packages/patroni/psycopg.py", line 103, in connect
    ret = _connect(*args, **kwargs)
  File "/usr/local/lib/python3.10/dist-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
psycopg2.OperationalError: connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: FATAL:  role "replicator" does not exist

2023-10-08 01:30:46,263 INFO: promoted self to leader by acquiring session lock
2023-10-08 01:30:46.264 UTC [5692] LOG:  received promote request
server promoting
2023-10-08 01:30:46.264 UTC [5692] LOG:  redo is not required
2023-10-08 01:30:46.287 UTC [5692] LOG:  selected new timeline ID: 12
2023-10-08 01:30:46.426 UTC [5692] LOG:  archive recovery complete
2023-10-08 01:30:46.472 UTC [5690] LOG:  database system is ready to accept connections
2023-10-08 01:30:47,293 INFO: no action. I am (dba7-pgsql1.ru-central1.internal), the leader with the lock
2023-10-08 01:30:51.904 UTC [5712] FATAL:  password authentication failed for user "replicator"
2023-10-08 01:30:51.904 UTC [5712] DETAIL:  Role "replicator" does not exist.
	Connection matched pg_hba.conf line 100: "host replication replicator 10.0.0.0/8 md5"
2023-10-08 01:30:57,309 INFO: no action. I am (dba7-pgsql1.ru-central1.internal), the leader with the lock
```
