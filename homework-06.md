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

