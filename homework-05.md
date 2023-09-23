## Углубленное изучение бэкапов и репликации 
### Cоздал виртуальные машины dba5-master и dba5-slave c Ubuntu 20.04 LTS (bionic) в ЯО 
#### Установил на нее PostgreSQL 15 
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'; sudo wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null; sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15
```
#### На первой машине (dba5-master)
```
master-user@dba5-master:~$ sudo su -
root@dba5-master:~# mkdir /archive
root@dba5-master:~# chown -R postgres:postgres /archive

root@dba5-master:~# su - postgres
postgres@dba5-master:~$ psql -c "CREATE USER replica WITH REPLICATION encrypted password 'test1234'"
CREATE ROLE

postgres@dba5-master:~$ psql -c "CREATE USER backup WITH REPLICATION encrypted password 'test1234'"
CREATE ROLE

postgres@dba5-master:~$ psql -c "create database sample"
CREATE DATABASE

postgres@dba5-master:~$ echo "listen_addresses = '10.128.0.10'" >> /etc/postgresql/15/main/postgresql.conf
postgres@dba5-master:~$ echo "host replication replica 10.128.0.10/24 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
postgres@dba5-master:~$ echo "archive_mode = on" >>  /etc/postgresql/15/main/postgresql.conf
postgres@dba5-master:~$ echo "archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'" >>  /etc/postgresql/15/main/postgresql.conf
postgres@dba5-master:~$ pg_ctlcluster 15 main stop
postgres@dba5-master:~$ /usr/lib/postgresql/15/bin/pg_checksums -D /var/lib/postgresql/15/main --enable
postgres@dba5-master:~$ pg_ctlcluster 15 main start
postgres@dba5-master:~$
logout
```

#### На второй машине (dba5-slave)
```
replica-user@dba5-replica:~$ sudo su -
root@dba5-replica:~# service postgresql stop

root@dba5-replica:~# mkdir /archive
root@dba5-replica:~# chown -R postgres:postgres /archive

root@dba5-replica:~# echo "host replication replica 10.128.0.10/24 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
root@dba5-replica:~# echo "archive_mode = on" >>  /etc/postgresql/15/main/postgresql.conf
root@dba5-replica:~# echo "archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'" >>  /etc/postgresql/15/main/postgresql.conf

root@dba5-replica:~# rm -rf /var/lib/postgresql/15/main

root@dba5-replica:~# pg_basebackup --host=10.128.0.10 --port=5432 --username=replica --pgdata=/var/lib/postgresql/15/main/ --progress --write-recovery-conf --create-slot --slot=replica2
Password:
30506/30506 kB (100%), 1/1 tablespace

root@dba5-replica:~# chown -R postgres:postgres /var/lib/postgresql/15/main

root@dba5-replica:~# service postgresql start

sudo sh -c 'echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" > /etc/apt/sources.list.d/pg_probackup.list' && sudo wget -O - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP | sudo apt-key add - && sudo apt-get update

sudo DEBIAN_FRONTEND=noninteractive apt install pg-probackup-15 pg-probackup-15-dbg postgresql-contrib postgresql-15-pg-checksums -y

root@dba5-replica:~# sudo rm -rf /home/backups && sudo mkdir /home/backups && sudo chown -R postgres:postgres /home/backups

root@dba5-replica:~# sudo su - postgres
postgres@dba5-replica:~$ echo "BACKUP_PATH=/home/backups/">>~/.bashrc
postgres@dba5-replica:~$ echo "export BACKUP_PATH">>~/.bashrc
postgres@dba5-replica:~$ cd ~/; . .bashrc
```

#### На первой машине (dba5-master)
```
root@dba5-master:~# su - postgres
postgres@dba5-master:~$ psql
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

postgres=# BEGIN;
--CREATE ROLE backup WITH LOGIN;
GRANT USAGE ON SCHEMA pg_catalog TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
COMMIT;

ALTER ROLE backup WITH REPLICATION;
BEGIN
CREATE ROLE
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
GRANT
COMMIT
ALTER ROLE
postgres=#
\q
```

#### На второй машине (dba5-slave)
```
postgres@dba5-replica:~$ pg_probackup-15 init
INFO: Backup catalog '/home/backups' successfully initialized

postgres@dba5-replica:~$ cd $BACKUP_PATH; ls -la
total 16
drwxr-xr-x 4 postgres postgres 4096 Sep 21 17:02 .
drwxr-xr-x 4 root     root     4096 Sep 21 16:49 ..
drwx------ 2 postgres postgres 4096 Sep 21 17:02 backups
drwx------ 2 postgres postgres 4096 Sep 21 17:02 wal

postgres@dba5-replica:/home/backups$ pg_probackup-15 add-instance --instance 'main' -D /var/lib/postgresql/15/main
INFO: Instance 'main' successfully initialized


postgres@dba5-replica:/home/backups$ pg_probackup-15 show-config --instance main
# Backup instance information
pgdata = /var/lib/postgresql/15/main
system-identifier = 7281312430595329532
xlog-seg-size = 16777216
# Connection parameters
pgdatabase = postgres
# Replica parameters
replica-timeout = 5min
# Archive parameters
archive-timeout = 5min
# Logging parameters
log-level-console = INFO
log-level-file = OFF
log-format-console = PLAIN
log-format-file = PLAIN
log-filename = pg_probackup.log
log-rotation-size = 0TB
log-rotation-age = 0d
# Retention parameters
retention-redundancy = 0
retention-window = 0
wal-depth = 0
# Compression parameters
compress-algorithm = none
compress-level = 1
# Remote access parameters
remote-proto = ssh

root@dba5-replica:~# su - postgres
postgres@dba5-replica:~$ vi ./.pgpass
postgres@dba5-replica:~$ cat ~/.pgpass
10.128.0.10:5432:replication:backup:test1234
10.128.0.10:5432:replication:replica:test1234
```

#### Снимаю бекап со второй машины
```
postgres@dba5-replica:~$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: S1GOPM, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
WARNING: password file "/var/lib/postgresql/.pgpass" has group or world access; permissions should be u=rw (0600) or less
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Backup S1GOPM is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
WARNING: password file "/var/lib/postgresql/.pgpass" has group or world access; permissions should be u=rw (0600) or less
INFO: Wait for WAL segment /home/backups/backups/main/S1GOPM/database/pg_wal/000000010000000000000002 to be streamed
INFO: PGDATA size: 29MB
INFO: Current Start LSN: 0/2000028, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 0
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/3021BE8
WARNING: Could not read WAL record at 0/3021BE8: invalid record length at 0/3021BE8: wanted 24, got 0
INFO: Wait for LSN 0/3021BE8 in streamed WAL segment /home/backups/backups/main/S1GOPM/database/pg_wal/000000010000000000000003
WARNING: Could not read WAL record at 0/3021BE8: invalid record length at 0/3021BE8: wanted 24, got 0
WARNING: Could not read WAL record at 0/3021BE8: invalid record length at 0/3021BE8: wanted 24, got 0
INFO: Getting the Recovery Time from WAL
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 17s
INFO: Validating backup S1GOPM
INFO: Backup S1GOPM data files are valid
INFO: Backup S1GOPM resident size: 61MB
INFO: Backup S1GOPM completed
```

#### Проблема: при повторном выполнении комманды начинает сыпать ворнингами:
```
pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: S1GOS3, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
WARNING: password file "/var/lib/postgresql/.pgpass" has group or world access; permissions should be u=rw (0600) or less
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
INFO: Backup S1GOS3 is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
WARNING: password file "/var/lib/postgresql/.pgpass" has group or world access; permissions should be u=rw (0600) or less


INFO: Wait for WAL segment /home/backups/backups/main/S1GOS3/database/pg_wal/000000010000000000000003 to be streamed
INFO: PGDATA size: 29MB
INFO: Current Start LSN: 0/3021BE8, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 0
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: stop_lsn: 0/3021CD0
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
INFO: Wait for LSN 0/3021CD0 in streamed WAL segment /home/backups/backups/main/S1GOS3/database/pg_wal/000000010000000000000003
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
WARNING: Could not read WAL record at 0/3021CD0: invalid record length at 0/3021CD0: wanted 24, got 0
```
