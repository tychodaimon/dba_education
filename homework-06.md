# Кластер Patroni on-premise

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
#namespace: /service/
name: dba7-pgsql1

restapi:
  listen: 10.128.0.42:8008
  connect_address: 10.128.0.42:8008
#  certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#  keyfile: /etc/ssl/private/ssl-cert-snakeoil.key
#  authentication:
#    username: username
#    password: password

# ctl:
#   insecure: false # Allow connections to SSL sites without certs
#   certfile: /etc/ssl/certs/ssl-cert-snakeoil.pem
#   cacert: /etc/ssl/certs/ssl-cacert-snakeoil.pem

etcd:
  #Provide host to do the initial discovery of the cluster topology:
  #host: 127.0.0.1:2379
  #Or use "hosts" to provide multiple endpoints
  #Could be a comma separated string:
  hosts: dba7-etcd1.ru-central1.internal:2379,dba7-etcd2.ru-central1.internal:2379,dba7-etcd3.ru-central1.internal:2379
  #or an actual yaml list:
  #hosts:
  #- host1:port1
  #- host2:port2
  #Once discovery is complete Patroni will use the list of advertised clientURLs
  #It is possible to change this behavior through by setting:
  #use_proxies: true

#raft:
#  data_dir: .
#  self_addr: 127.0.0.1:2222
#  partner_addrs:
#  - 127.0.0.1:2223
#  - 127.0.0.1:2224

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  # and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
#    master_start_timeout: 300
#    synchronous_mode: false
    #standby_cluster:
      #host: 127.0.0.1
      #port: 1111
      #primary_slot_name: patroni
    postgresql:
      use_pg_rewind: true
#      use_slots: true
      parameters:
#        wal_level: hot_standby
#        hot_standby: "on"
#        max_connections: 100
#        max_worker_processes: 8
#        wal_keep_segments: 8
#        max_wal_senders: 10
#        max_replication_slots: 10
#        max_prepared_transactions: 0
#        max_locks_per_transaction: 64
#        wal_log_hints: "on"
#        track_commit_timestamp: "off"
#        archive_mode: "on"
#        archive_timeout: 1800s
#        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
#      recovery_conf:
#        restore_command: cp ../wal_archive/%f %p

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  # For kerberos gss based connectivity (discard @.*$)
  #- host replication replicator 127.0.0.1/32 gss include_realm=0
  #- host all all 0.0.0.0/0 gss include_realm=0
  - host replication replicator 10.0.0.0/8 md5
  - host all all 10.0.0.0/8 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Additional script to be launched after initial cluster creation (will be passed the connection URL as parameter)
# post_init: /usr/local/bin/setup_cluster.sh

  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin_321
      options:
        - createrole
        - createdb
    replicator:
      password: rep-pass_321
      options:
        - createrole
        - createdb
postgresql:
  listen: 127.0.0.1, 10.128.0.42:5432
  connect_address: 10.128.0.42:5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  config_dir: /var/lib/postgresql/14/main
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: zalando_321
    rewind:  # Has no effect on postgres 10 and lower
      username: rewind_user
      password: rewind_password_321
  # Server side kerberos spn
 # krbsrvname: postgres
  parameters:
    # Fully qualified kerberos ticket file for the running user
    # same as KRB5CCNAME used by the GSS
  # krb_server_keyfile: /var/spool/keytabs/postgres
    unix_socket_directories: '.'
  # Additional fencing script executed after acquiring the leader lock but before promoting the replica
  #pre_promote: /path/to/pre_promote.sh

watchdog:
  mode: automatic # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

#### Запускаю
```
postgres@dba7-pgsql1:~$ patroni /etc/patroni.yml
2023-10-08 23:49:20,257 INFO: Selected new etcd server http://dba7-etcd1.ru-central1.internal:2379
2023-10-08 23:49:20,263 INFO: No PostgreSQL configuration items changed, nothing to reload.
2023-10-08 23:49:20,274 WARNING: Postgresql is not running.
2023-10-08 23:49:20,275 INFO: Lock owner: None; I am dba7-pgsql1
2023-10-08 23:49:20,276 INFO: pg_controldata:
  pg_control version number: 1300
  Catalog version number: 202107181
  Database system identifier: 7287737210240395647
  Database cluster state: shut down
  pg_control last modified: Sun Oct  8 23:49:07 2023
  Latest checkpoint location: 0/194E868
  Latest checkpoint's REDO location: 0/194E868
  Latest checkpoint's REDO WAL file: 000000030000000000000001
  Latest checkpoint's TimeLineID: 3
  Latest checkpoint's PrevTimeLineID: 3
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
  Time of latest checkpoint: Sun Oct  8 23:49:07 2023
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
  Mock authentication nonce: ea256115b484750eaa070bbdaec5d3880239c49f9962e2b0eca07a1670d69512

2023-10-08 23:49:20,281 INFO: Lock owner: None; I am dba7-pgsql1
2023-10-08 23:49:20,287 INFO: starting as a secondary
2023-10-08 23:49:20,701 INFO: postmaster pid=6783
localhost:5432 - no response
2023-10-08 23:49:20.721 UTC [6783] LOG:  starting PostgreSQL 14.9 (Ubuntu 14.9-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2023-10-08 23:49:20.721 UTC [6783] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2023-10-08 23:49:20.729 UTC [6783] LOG:  listening on IPv4 address "10.128.0.42", port 5432
2023-10-08 23:49:20.741 UTC [6783] LOG:  listening on Unix socket "./.s.PGSQL.5432"
2023-10-08 23:49:20.753 UTC [6785] LOG:  database system was shut down at 2023-10-08 23:49:07 UTC
2023-10-08 23:49:20.753 UTC [6785] WARNING:  specified neither primary_conninfo nor restore_command
2023-10-08 23:49:20.753 UTC [6785] HINT:  The database server will regularly poll the pg_wal subdirectory to check for files placed there.
2023-10-08 23:49:20.753 UTC [6785] LOG:  entering standby mode
2023-10-08 23:49:20.763 UTC [6785] LOG:  consistent recovery state reached at 0/194E8E0
2023-10-08 23:49:20.763 UTC [6785] LOG:  invalid record length at 0/194E8E0: wanted 24, got 0
2023-10-08 23:49:20.764 UTC [6783] LOG:  database system is ready to accept read-only connections
localhost:5432 - accepting connections
localhost:5432 - accepting connections
2023-10-08 23:49:21,724 INFO: establishing a new patroni connection to the postgres cluster
2023-10-08 23:49:21,733 WARNING: Could not activate Linux watchdog device: Can't open watchdog device: [Errno 2] No such file or directory: '/dev/watchdog'
2023-10-08 23:49:21.735 UTC [6795] FATAL:  role "replicator" does not exist
2023-10-08 23:49:21,735 ERROR: Can not fetch local timeline and lsn from replication connection
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

2023-10-08 23:49:21,744 INFO: promoted self to leader by acquiring session lock
2023-10-08 23:49:21.745 UTC [6785] LOG:  received promote request
2023-10-08 23:49:21.745 UTC [6785] LOG:  redo is not required
server promoting
2023-10-08 23:49:21.752 UTC [6785] LOG:  selected new timeline ID: 4
2023-10-08 23:49:21.856 UTC [6785] LOG:  archive recovery complete
2023-10-08 23:49:21.895 UTC [6783] LOG:  database system is ready to accept connections
2023-10-08 23:49:22,771 INFO: no action. I am (dba7-pgsql1), the leader with the lock
2023-10-08 23:49:32,763 INFO: no action. I am (dba7-pgsql1), the leader with the lock
2023-10-08 23:49:42,764 INFO: no action. I am (dba7-pgsql1), the leader with the lock
```

#### Создаю недостающих пользователей
```
root@dba7-pgsql1:~# psql -h 127.0.0.1 -U postgres
postgres=# CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'rep-pass_321';
CREATE ROLE
postgres=# CREATE ROLE rewind_user WITH REPLICATION LOGIN PASSWORD 'rewind_password_321';
CREATE ROLE
```

#### Сбрасываю состояние кластера
```
root@dba7-pgsql1:~# patronictl -c /etc/patroni.yml remove patroni
```

#### Дополнительно фикшу
>WARNING: Could not activate Linux watchdog device: Can't open watchdog device: [Errno 2] No such file or directory: '/dev/watchdog'
```
root@dba7-pgsql1:~# modprobe softdog
root@dba7-pgsql1:~# chown postgres:postgres /dev/watchdog
```

#### Повторно запускаю
```
postgres@dba7-pgsql1:~$ patroni /etc/patroni.yml
2023-10-09 00:21:15,059 INFO: Selected new etcd server http://dba7-etcd1.ru-central1.internal:2379
2023-10-09 00:21:15,068 INFO: No PostgreSQL configuration items changed, nothing to reload.
2023-10-09 00:21:15,073 WARNING: Postgresql is not running.
2023-10-09 00:21:15,073 INFO: Lock owner: None; I am dba7-pgsql1
2023-10-09 00:21:15,074 INFO: pg_controldata:
  pg_control version number: 1300
  Catalog version number: 202107181
  Database system identifier: 7287737210240395647
  Database cluster state: shut down
  pg_control last modified: Mon Oct  9 00:21:12 2023
  Latest checkpoint location: 0/19501B0
  Latest checkpoint's REDO location: 0/19501B0
  Latest checkpoint's REDO WAL file: 0000000A0000000000000001
  Latest checkpoint's TimeLineID: 10
  Latest checkpoint's PrevTimeLineID: 10
  Latest checkpoint's full_page_writes: on
  Latest checkpoint's NextXID: 0:736
  Latest checkpoint's NextOID: 16386
  Latest checkpoint's NextMultiXactId: 1
  Latest checkpoint's NextMultiOffset: 0
  Latest checkpoint's oldestXID: 727
  Latest checkpoint's oldestXID's DB: 1
  Latest checkpoint's oldestActiveXID: 0
  Latest checkpoint's oldestMultiXid: 1
  Latest checkpoint's oldestMulti's DB: 1
  Latest checkpoint's oldestCommitTsXid: 0
  Latest checkpoint's newestCommitTsXid: 0
  Time of latest checkpoint: Mon Oct  9 00:21:12 2023
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
  Mock authentication nonce: ea256115b484750eaa070bbdaec5d3880239c49f9962e2b0eca07a1670d69512

2023-10-09 00:21:15,080 INFO: Lock owner: None; I am dba7-pgsql1
2023-10-09 00:21:15,107 INFO: starting as a secondary
2023-10-09 00:21:15,532 INFO: postmaster pid=7788
localhost:5432 - no response
2023-10-09 00:21:15.550 UTC [7788] LOG:  starting PostgreSQL 14.9 (Ubuntu 14.9-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2023-10-09 00:21:15.550 UTC [7788] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2023-10-09 00:21:15.553 UTC [7788] LOG:  listening on IPv4 address "10.128.0.42", port 5432
2023-10-09 00:21:15.558 UTC [7788] LOG:  listening on Unix socket "./.s.PGSQL.5432"
2023-10-09 00:21:15.569 UTC [7790] LOG:  database system was shut down at 2023-10-09 00:21:12 UTC
2023-10-09 00:21:15.569 UTC [7790] WARNING:  specified neither primary_conninfo nor restore_command
2023-10-09 00:21:15.569 UTC [7790] HINT:  The database server will regularly poll the pg_wal subdirectory to check for files placed there.
2023-10-09 00:21:15.569 UTC [7790] LOG:  entering standby mode
2023-10-09 00:21:15.575 UTC [7790] LOG:  consistent recovery state reached at 0/1950228
2023-10-09 00:21:15.576 UTC [7790] LOG:  invalid record length at 0/1950228: wanted 24, got 0
2023-10-09 00:21:15.577 UTC [7788] LOG:  database system is ready to accept read-only connections
localhost:5432 - accepting connections
localhost:5432 - accepting connections
2023-10-09 00:21:16,556 INFO: establishing a new patroni connection to the postgres cluster
2023-10-09 00:21:16,563 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
2023-10-09 00:21:16,571 INFO: promoted self to leader by acquiring session lock
server promoting
2023-10-09 00:21:16.572 UTC [7790] LOG:  received promote request
2023-10-09 00:21:16.572 UTC [7790] LOG:  redo is not required
2023-10-09 00:21:16.617 UTC [7790] LOG:  selected new timeline ID: 11
2023-10-09 00:21:16.724 UTC [7790] LOG:  archive recovery complete
2023-10-09 00:21:16.744 UTC [7788] LOG:  database system is ready to accept connections
2023-10-09 00:21:17,592 INFO: no action. I am (dba7-pgsql1), the leader with the lock
2023-10-09 00:21:27,587 INFO: no action. I am (dba7-pgsql1), the leader with the lock
````

#### На других тачках
```
postgres@dba7-pgsql2:~$ patroni /etc/patroni.yml
2023-10-09 00:22:13,533 INFO: Selected new etcd server http://dba7-etcd2.ru-central1.internal:2379
2023-10-09 00:22:13,542 INFO: No PostgreSQL configuration items changed, nothing to reload.
2023-10-09 00:22:13,550 INFO: Lock owner: dba7-pgsql1; I am dba7-pgsql2
2023-10-09 00:22:13,554 INFO: trying to bootstrap from leader 'dba7-pgsql1'
WARNING:  skipping special file "./.s.PGSQL.5432"
WARNING:  skipping special file "./.s.PGSQL.5432"
2023-10-09 00:22:17,653 INFO: replica has been created using basebackup
2023-10-09 00:22:17,654 INFO: bootstrapped from leader 'dba7-pgsql1'
2023-10-09 00:22:18,551 INFO: postmaster pid=3834
localhost:5432 - no response
2023-10-09 00:22:18.620 UTC [3834] LOG:  starting PostgreSQL 14.9 (Ubuntu 14.9-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2023-10-09 00:22:18.620 UTC [3834] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2023-10-09 00:22:18.623 UTC [3834] LOG:  listening on IPv4 address "10.128.0.6", port 5432
2023-10-09 00:22:18.627 UTC [3834] LOG:  listening on Unix socket "./.s.PGSQL.5432"
2023-10-09 00:22:18.639 UTC [3836] LOG:  database system was interrupted; last known up at 2023-10-09 00:22:14 UTC
2023-10-09 00:22:18.646 UTC [3836] LOG:  entering standby mode
2023-10-09 00:22:18.660 UTC [3836] LOG:  redo starts at 0/2000028
2023-10-09 00:22:18.662 UTC [3836] LOG:  consistent recovery state reached at 0/2000138
2023-10-09 00:22:18.662 UTC [3834] LOG:  database system is ready to accept read-only connections
2023-10-09 00:22:18.690 UTC [3840] LOG:  started streaming WAL from primary at 0/3000000 on timeline 11
localhost:5432 - accepting connections
localhost:5432 - accepting connections
2023-10-09 00:22:19,592 INFO: Lock owner: dba7-pgsql1; I am dba7-pgsql2
2023-10-09 00:22:19,592 INFO: establishing a new patroni connection to the postgres cluster
2023-10-09 00:22:19,651 INFO: no action. I am (dba7-pgsql2), a secondary, and following a leader (dba7-pgsql1)
2023-10-09 00:22:27,624 INFO: no action. I am (dba7-pgsql2), a secondary, and following a leader (dba7-pgsql1)
```
и
```
postgres@dba7-pgsql3:~$ patroni /etc/patroni.yml
2023-10-09 00:22:42,736 INFO: Selected new etcd server http://dba7-etcd2.ru-central1.internal:2379
2023-10-09 00:22:42,746 INFO: No PostgreSQL configuration items changed, nothing to reload.
2023-10-09 00:22:42,752 INFO: Lock owner: dba7-pgsql1; I am dba7-pgsql3
2023-10-09 00:22:42,756 INFO: trying to bootstrap from leader 'dba7-pgsql1'
WARNING:  skipping special file "./.s.PGSQL.5432"
WARNING:  skipping special file "./.s.PGSQL.5432"
2023-10-09 00:22:47,519 INFO: replica has been created using basebackup
2023-10-09 00:22:47,520 INFO: bootstrapped from leader 'dba7-pgsql1'
2023-10-09 00:22:48,500 INFO: postmaster pid=1833
localhost:5432 - no response
2023-10-09 00:22:48.541 UTC [1833] LOG:  starting PostgreSQL 14.9 (Ubuntu 14.9-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2023-10-09 00:22:48.541 UTC [1833] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2023-10-09 00:22:48.548 UTC [1833] LOG:  listening on IPv4 address "10.128.0.26", port 5432
2023-10-09 00:22:48.556 UTC [1833] LOG:  listening on Unix socket "./.s.PGSQL.5432"
2023-10-09 00:22:48.564 UTC [1835] LOG:  database system was interrupted; last known up at 2023-10-09 00:22:43 UTC
2023-10-09 00:22:48.572 UTC [1835] LOG:  entering standby mode
2023-10-09 00:22:48.579 UTC [1835] LOG:  redo starts at 0/4000028
2023-10-09 00:22:48.581 UTC [1835] LOG:  consistent recovery state reached at 0/4000138
2023-10-09 00:22:48.581 UTC [1833] LOG:  database system is ready to accept read-only connections
2023-10-09 00:22:48.599 UTC [1839] LOG:  started streaming WAL from primary at 0/5000000 on timeline 11
localhost:5432 - accepting connections
localhost:5432 - accepting connections
2023-10-09 00:22:49,537 INFO: Lock owner: dba7-pgsql1; I am dba7-pgsql3
2023-10-09 00:22:49,537 INFO: establishing a new patroni connection to the postgres cluster
2023-10-09 00:22:49,581 INFO: no action. I am (dba7-pgsql3), a secondary, and following a leader (dba7-pgsql1)
2023-10-09 00:22:57,608 INFO: no action. I am (dba7-pgsql3), a secondary, and following a leader (dba7-pgsql1)
```

#### На всех постгресах запускаю в виде службы и включаю автозапуск
```
root@dba7-pgsql1:~# systemctl enable patroni && sudo systemctl start patroni
```

#### Проверяю кластер
```
dba@dba7-pgsql1:~$ patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7287737210240395647) ---------+----+-----------+
| Member      | Host        | Role    | State     | TL | Lag in MB |
+-------------+-------------+---------+-----------+----+-----------+
| dba7-pgsql1 | 10.128.0.42 | Leader  | running   | 19 |           |
| dba7-pgsql2 | 10.128.0.6  | Replica | streaming | 19 |         0 |
| dba7-pgsql3 | 10.128.0.26 | Replica | streaming | 19 |         0 |
+-------------+-------------+---------+-----------+----+-----------+
```

#### На всех постгресах ставлю pgBouncer
```
dba@dba7-pgsql1:~$ sudo apt install -y pgbouncer

root@dba7-pgsql1:~# vi /etc/pgbouncer/pgbouncer.ini

root@dba7-pgsql1:~# cat /etc/pgbouncer/pgbouncer.ini
;;;
;;; PgBouncer configuration file
;;;

;; database name = connect string
;;
;; connect string params:
;;   dbname= host= port= user= password= auth_user=
;;   client_encoding= datestyle= timezone=
;;   pool_size= reserve_pool= max_db_connections=
;;   pool_mode= connect_query= application_name=
[databases]

;; foodb over Unix socket
;foodb =

;; redirect bardb to bazdb on localhost
;bardb = host=localhost dbname=bazdb

;; access to dest database will go with single user
;forcedb = host=localhost port=300 user=baz password=foo client_encoding=UNICODE datestyle=ISO connect_query='SELECT 1'

;; use custom pool sizes
;nondefaultdb = pool_size=50 reserve_pool=10

;; use auth_user with auth_query if user not present in auth_file
;; auth_user must exist in auth_file
; foodb = auth_user=bar

;; run auth_query on a specific database.
; bardb = auth_dbname=foo

;; fallback connect string
;* = host=testserver

;; User-specific configuration
[users]

;user1 = pool_mode=transaction max_user_connections=10

;; Configuration section
[pgbouncer]

;;;
;;; Administrative settings
;;;

logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid

;;;
;;; Where to wait for clients
;;;

;; IP address or * which means all IPs
listen_addr = localhost
listen_port = 6432

;; Unix socket is also used for -R.
;; On Debian it should be /var/run/postgresql
;unix_socket_dir = /tmp
;unix_socket_mode = 0777
;unix_socket_group =
unix_socket_dir = /var/run/postgresql

;; The peer id used to identify this pgbouncer process in a group of pgbouncer
;; processes that are peered together. When set to 0 pgbouncer peering is disabled
;peer_id = 0

;;;
;;; TLS settings for accepting clients
;;;

;; disable, allow, require, verify-ca, verify-full
;client_tls_sslmode = disable

;; Path to file that contains trusted CA certs
;client_tls_ca_file = <system default>

;; Private key and cert to present to clients.
;; Required for accepting TLS connections from clients.
;client_tls_key_file =
;client_tls_cert_file =

;; fast, normal, secure, legacy, <ciphersuite string>
;client_tls_ciphers = fast

;; all, secure, tlsv1.0, tlsv1.1, tlsv1.2, tlsv1.3
;client_tls_protocols = secure

;; none, auto, legacy
;client_tls_dheparams = auto

;; none, auto, <curve name>
;client_tls_ecdhcurve = auto

;;;
;;; TLS settings for connecting to backend databases
;;;

;; disable, allow, require, verify-ca, verify-full
;server_tls_sslmode = disable

;; Path to that contains trusted CA certs
;server_tls_ca_file = <system default>

;; Private key and cert to present to backend.
;; Needed only if backend server require client cert.
;server_tls_key_file =
;server_tls_cert_file =

;; all, secure, tlsv1.0, tlsv1.1, tlsv1.2, tlsv1.3
;server_tls_protocols = secure

;; fast, normal, secure, legacy, <ciphersuite string>
;server_tls_ciphers = fast

;;;
;;; Authentication settings
;;;

;; any, trust, plain, md5, cert, hba, pam
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

;; Path to HBA-style auth config
;auth_hba_file =

;; Query to use to fetch password from database.  Result
;; must have 2 columns - username and password hash.
;auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1

;; Authentication database that can be set globally to run "auth_query".
;auth_dbname =

;;;
;;; Users allowed into database 'pgbouncer'
;;;

;; comma-separated list of users who are allowed to change settings
;admin_users = user2, someadmin, otheradmin

;; comma-separated list of users who are just allowed to use SHOW command
;stats_users = stats, root

;;;
;;; Pooler personality questions
;;;

;; When server connection is released back to pool:
;;   session      - after client disconnects (default)
;;   transaction  - after transaction finishes
;;   statement    - after statement finishes
;pool_mode = session

;; Query for cleaning connection immediately after releasing from
;; client.  No need to put ROLLBACK here, pgbouncer does not reuse
;; connections where transaction is left open.
;server_reset_query = DISCARD ALL

;; Whether server_reset_query should run in all pooling modes.  If it
;; is off, server_reset_query is used only for session-pooling.
;server_reset_query_always = 0

;; Comma-separated list of parameters to track per client.  The
;; Postgres parameters listed here will be cached per client by
;; pgbouncer and restored in server everytime the client runs a query.
;track_extra_parameters = IntervalStyle

;; Comma-separated list of parameters to ignore when given in startup
;; packet.  Newer JDBC versions require the extra_float_digits here.
;ignore_startup_parameters = extra_float_digits

;; When taking idle server into use, this query is run first.
;server_check_query = select 1

;; If server was used more recently that this many seconds ago,
; skip the check query.  Value 0 may or may not run in immediately.
;server_check_delay = 30

;; Close servers in session pooling mode after a RECONNECT, RELOAD,
;; etc. when they are idle instead of at the end of the session.
;server_fast_close = 0

;; Use <appname - host> as application_name on server.
;application_name_add_host = 0

;; Period for updating aggregated stats.
;stats_period = 60

;;;
;;; Connection limits
;;;

;; Total number of clients that can connect
;max_client_conn = 100

;; Default pool size.  20 is good number when transaction pooling
;; is in use, in session pooling it needs to be the number of
;; max clients you want to handle at any moment
;default_pool_size = 20

;; Minimum number of server connections to keep in pool.
;min_pool_size = 0

; how many additional connection to allow in case of trouble
;reserve_pool_size = 0

;; If a clients needs to wait more than this many seconds, use reserve
;; pool.
;reserve_pool_timeout = 5

;; Maximum number of server connections for a database
;max_db_connections = 0

;; Maximum number of server connections for a user
;max_user_connections = 0

;; If off, then server connections are reused in LIFO manner
;server_round_robin = 0

;;;
;;; Logging
;;;

;; Syslog settings
;syslog = 0
;syslog_facility = daemon
;syslog_ident = pgbouncer

;; log if client connects or server connection is made
;log_connections = 1

;; log if and why connection was closed
;log_disconnections = 1

;; log error messages pooler sends to clients
;log_pooler_errors = 1

;; write aggregated stats into log
;log_stats = 1

;; Logging verbosity.  Same as -v switch on command line.
;verbose = 0

;;;
;;; Timeouts
;;;

;; Close server connection if its been connected longer.
;server_lifetime = 3600

;; Close server connection if its not been used in this time.  Allows
;; to clean unnecessary connections from pool after peak.
;server_idle_timeout = 600

;; Cancel connection attempt if server does not answer takes longer.
;server_connect_timeout = 15

;; If server login failed (server_connect_timeout or auth failure)
;; then wait this many second before trying again.
;server_login_retry = 15

;; Dangerous.  Server connection is closed if query does not return in
;; this time.  Should be used to survive network problems, _not_ as
;; statement_timeout. (default: 0)
;query_timeout = 0

;; Dangerous.  Client connection is closed if the query is not
;; assigned to a server in this time.  Should be used to limit the
;; number of queued queries in case of a database or network
;; failure. (default: 120)
;query_wait_timeout = 120

;; Dangerous.  Client connection is closed if the cancellation request
;; is not assigned to a server in this time.  Should be used to limit
;; the time a client application blocks on a queued cancel request in
;; case of a database or network failure. (default: 120)
;cancel_wait_timeout = 10

;; Dangerous.  Client connection is closed if no activity in this
;; time.  Should be used to survive network problems. (default: 0)
;client_idle_timeout = 0

;; Disconnect clients who have not managed to log in after connecting
;; in this many seconds.
;client_login_timeout = 60

;; Clean automatically created database entries (via "*") if they stay
;; unused in this many seconds.
; autodb_idle_timeout = 3600

;; Close connections which are in "IDLE in transaction" state longer
;; than this many seconds.
;idle_transaction_timeout = 0

;; How long SUSPEND/-R waits for buffer flush before closing
;; connection.
;suspend_timeout = 10

;;;
;;; Low-level tuning options
;;;

;; buffer for streaming packets
;pkt_buf = 4096

;; man 2 listen
;listen_backlog = 128

;; Max number pkt_buf to process in one event loop.
;sbuf_loopcnt = 5

;; Maximum PostgreSQL protocol packet size.
;max_packet_size = 2147483647

;; Set SO_REUSEPORT socket option
;so_reuseport = 0

;; networking options, for info: man 7 tcp

;; Linux: Notify program about new connection only if there is also
;; data received.  (Seconds to wait.)  On Linux the default is 45, on
;; other OS'es 0.
;tcp_defer_accept = 0

;; In-kernel buffer size (Linux default: 4096)
;tcp_socket_buffer = 0

;; whether tcp keepalive should be turned on (0/1)
;tcp_keepalive = 1

;; The following options are Linux-specific.  They also require
;; tcp_keepalive=1.

;; Count of keepalive packets
;tcp_keepcnt = 0

;; How long the connection can be idle before sending keepalive
;; packets
;tcp_keepidle = 0

;; The time between individual keepalive probes
;tcp_keepintvl = 0

;; How long may transmitted data remain unacknowledged before TCP
;; connection is closed (in milliseconds)
;tcp_user_timeout = 0

;; DNS lookup caching time
;dns_max_ttl = 15

;; DNS zone SOA lookup period
;dns_zone_check_period = 0

;; DNS negative result caching time
;dns_nxdomain_ttl = 15

;; Custom resolv.conf file, to set custom DNS servers or other options
;; (default: empty = use OS settings)
;resolv_conf = /etc/pgbouncer/resolv.conf

;;;
;;; Random stuff
;;;

;; Hackish security feature.  Helps against SQL injection: when PQexec
;; is disabled, multi-statement cannot be made.
;disable_pqexec = 0

;; Config file to use for next RELOAD/SIGHUP
;; By default contains config file from command line.
;conffile

;; Windows service name to register as.  job_name is alias for
;; service_name, used by some Skytools scripts.
;service_name = pgbouncer
;job_name = pgbouncer

;; Read additional config from other file
;%include /etc/pgbouncer/pgbouncer-other.ini

[databases]
otus = host=127.0.0.1 port=5432 dbname=otus
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
admin_users = admindb, postgres
```

#### Проверяю что все ок и создаю тестовую базу данных
```
root@dba7-pgsql1:~# sudo -u postgres psql -h localhost -c "CREATE DATABASE otus;"
CREATE DATABASE
```

#### Доавляю запись в pgpass
```
postgres@dba7-pgsql1:~$ echo "localhost:5432:postgres:postgres:zalando_321">>~/.pgpass
postgres@dba7-pgsql1:~$ echo "localhost:5432:otus:postgres:zalando_321">>~/.pgpass
postgres@dba7-pgsql1:~$ chmod 600 ~/.pgpass


postgres@dba7-pgsql1:~$ psql -h localhost
localhost postgres@postgres=# create user admindb with password 'root123';

localhost postgres@postgres=# \du
                                    List of roles
  Role name  |                         Attributes                         | Member of
-------------+------------------------------------------------------------+-----------
 admindb     |                                                            | {}
 postgres    | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 replicator  | Replication                                                | {}
 rewind_user | Replication                                                | {}

localhost postgres@postgres=# select usename,passwd from pg_shadow;
   usename   |                                                                passwd
-------------+---------------------------------------------------------------------------------------------------------------------------------------
 postgres    | SCRAM-SHA-256$4096:732gAsIxkd+rY2rL91/2RQ==$yGy5yKxWYjq8pPL5wYIDCrkUeyRWh+MGsAsegrpDnkM=:0kTaQr3WR3bWxcdIkx3ioM6syI3NsrLNuWQV5+eYLyA=
 replicator  | SCRAM-SHA-256$4096:tYDzvixLzrQAcBw+e5nXQg==$ln7P0AfTjA+HLMqwgw+HOczdfsqBN0FnzG82bf0XIKQ=:Wh99wmw3z89LBSz+sD3BZLufMuefQkXQ8gN1BJTG8B8=
 rewind_user | SCRAM-SHA-256$4096:V+ybGF4WBqdtGIhXSZIe3w==$Y7cub1tP/OAom8Ingua9P0qo1JfZZD75x8WzaWkc2HA=:uxy8zfLndfw6bY3GRStTXFqQuv+ThtIwMJ4QzvPLQ8E=
 admindb     | SCRAM-SHA-256$4096:pqu59EyyDXREBgIGPLlYfw==$B7ckmWLf37ZU5R/darh/h2vWZbT471UoRTk/C5q6sgI=:aP2EXVg8VjHzqazNeiXaxeRULTRBSUqv5S0WQEyFUhA=
(4 rows)

root@dba7-pgsql1:~# vi /etc/pgbouncer/userlist.txt
root@dba7-pgsql1:~# cat /etc/pgbouncer/userlist.txt
"admindb" "d9cfab6a2f1a0eb0c037e605cd578025"
"postgres"  "SCRAM-SHA-256$4096:732gAsIxkd+rY2rL91/2RQ==$yGy5yKxWYjq8pPL5wYIDCrkUeyRWh+MGsAsegrpDnkM=:0kTaQr3WR3bWxcdIkx3ioM6syI3NsrLNuWQV5+eYLyA="
```

#### Проверяю
```
postgres@dba7-pgsql1:~$ psql -p 6432 pgbouncer -h localhost
Password for user postgres:
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1), server 1.20.1/bouncer)
Type "help" for help.

localhost postgres@pgbouncer=#
\q
postgres@dba7-pgsql1:~$ psql -p 6432 -h 127.0.0.1 -d otus -U postgres
Password for user postgres:
psql (14.9 (Ubuntu 14.9-1.pgdg22.04+1))
Type "help" for help.
```

#### Ставлю haproxy
```
root@dba7-haproxy1:~# sudo apt install -y --no-install-recommends software-properties-common && sudo add-apt-repository -y ppa:vbernat/haproxy-2.5 && sudo apt install -y haproxy=2.5.\*

root@dba7-haproxy1:~# vi /etc/haproxy/haproxy.cfg
root@dba7-haproxy1:~# cat /etc/haproxy/haproxy.cfg
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

listen postgres_write
    bind *:5432
    mode            tcp
    option httpchk
    http-check connect
    http-check send meth GET uri /master
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql1 10.128.0.42:6432 check port 8008
    server pgsql2 10.128.0.6:6432 check port 8008
    server pgsql3 10.128.0.26:6432 check port 8008

listen postgres_read
    bind *:5433
    mode            tcp
    http-check connect
    http-check send meth GET uri /replica
    http-check expect status 200
    default-server inter 10s fall 3 rise 3 on-marked-down shutdown-sessions
    server pgsql1 10.128.0.42:6432 check port 8008
    server pgsql2 10.128.0.6:6432 check port 8008
    server pgsql3 10.128.0.26:6432 check port 8008
```

#### Клиент для postgres
```
root@dba7-haproxy1:~# apt update && sudo apt upgrade -y && sudo apt install -y postgresql-client-common && sudo apt install postgresql-client -y
```

#### Проверяю
```
root@dba7-haproxy1:~# psql -h localhost -d otus -U postgres -p 5432
Password for user postgres:
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1))
Type "help" for help.

otus=#
```
