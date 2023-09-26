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

postgres@dba5-master:~$ psql -c "CREATE USER user_for_backup WITH REPLICATION encrypted password 'test1234'"
CREATE ROLE

postgres@dba5-master:~$ psql -c "create database sample"
CREATE DATABASE

postgres@dba5-master:~$ echo "listen_addresses = '10.128.0.10'" >> /etc/postgresql/15/main/postgresql.conf
postgres@dba5-master:~$ echo "host replication replica 10.128.0.10/24 scram-sha-256" >> /etc/postgresql/15/main/pg_hba.conf
postgres@dba5-master:~$ echo "archive_mode = on" >>  /etc/postgresql/15/main/postgresql.conf
postgres@dba5-master:~$ echo "archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'" >>  /etc/postgresql/15/main/postgresql.conf
ostgres@dba5-master:~$ echo "archive_timeout = 30" >> /etc/postgresql/15/main/postgresql.conf
postgres@dba5-master:~$ echo "checkpoint_timeout = 30" >> /etc/postgresql/15/main/postgresql.conf
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

root@dba5-replica:~# su - postgres
postgres@dba5-replica:~$ vi ~/.pgpass
postgres@dba5-replica:~$ chmod 0600 ~/.pgpass
postgres@dba5-replica:~$ cat ./.pgpass
10.128.0.34:5432:replication:user_for_backup:test1234
10.128.0.34:5432:replication:replica:test1234
127.0.0.1:5432:replication:user_for_backup:test1234

postgres@dba5-replica:~# pg_basebackup --host=10.128.0.10 --port=5432 --username=replica --pgdata=/var/lib/postgresql/15/main/ --progress --write-recovery-conf --create-slot --slot=replica2 --no-password
30506/30506 kB (100%), 1/1 tablespace

postgres@dba5-replica:~# chown -R postgres:postgres /var/lib/postgresql/15/main

postgres@dba5-replica:~#  pg_ctlcluster 15 main start

#### here we have replica working now:

postgres@dba5-slave:~$ psql sample
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

sample=#
\q

postgres@dba5-slave:~$
logout

#### setup backup routine

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
GRANT USAGE ON SCHEMA pg_catalog TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO user_for_backup;
COMMIT;

ALTER ROLE user_for_backup WITH REPLICATION;
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
postgres@dba5-master:~$ psql sample
psql (15.4 (Ubuntu 15.4-2.pgdg22.04+1))
Type "help" for help.

sample=# BEGIN;
GRANT USAGE ON SCHEMA pg_catalog TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO user_for_backup;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO user_for_backup;
COMMIT;
BEGIN
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
sample=#
\q
postgres@dba5-master:~$
```

#### На второй машине (dba5-slave)
```
postgres@dba5-replica:~$ pg_probackup-15 init
INFO: Backup catalog '/home/backups' successfully initialized

postgres@dba5-replica:~$ cd $BACKUP_PATH; ls -la
total 16
drwxr-xr-x 4 postgres postgres 4096 Sep 26 21:53 .
drwxr-xr-x 4 root     root     4096 Sep 26 21:46 ..
drwx------ 2 postgres postgres 4096 Sep 26 21:53 backups
drwx------ 2 postgres postgres 4096 Sep 26 21:53 wal

postgres@dba5-replica:/home/backups$ pg_probackup-15 add-instance --instance 'main' -D /var/lib/postgresql/15/main
INFO: Instance 'main' successfully initialized
```

#### Снимаю бекап со второй машины
```
postgres@dba5-slave:~$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot -h 127.0.0.1 -U user_for_backup -d sample -p 5432
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: S1M891, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
Password for user user_for_backup:
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Backup S1M891 is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/backups/backups/main/S1M891/database/pg_wal/00000001000000000000000C to be streamed
INFO: PGDATA size: 29MB
INFO: Current Start LSN: 0/C000028, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 0
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: Wait for WAL segment /home/backups/backups/main/S1M891/database/pg_wal/00000001000000000000000D to be streamed
INFO: stop_lsn: 0/D000000
INFO: Getting the Recovery Time from WAL
INFO: Failed to find Recovery Time in WAL, forced to trust current_timestamp
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 9s
INFO: Validating backup S1M891
INFO: Backup S1M891 data files are valid
INFO: Backup S1M891 resident size: 61MB
INFO: Backup S1M891 completed
postgres@dba5-slave:~$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot -h 127.0.0.1 -U user_for_backup -d sample -p 5432
INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: S1M89T, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
Password for user user_for_backup:
INFO: This PostgreSQL instance was initialized with data block checksums. Data block corruption will be detected
INFO: Backup S1M89T is going to be taken from standby
INFO: Database backup start
INFO: wait for pg_backup_start()
INFO: Wait for WAL segment /home/backups/backups/main/S1M89T/database/pg_wal/00000001000000000000000D to be streamed
INFO: PGDATA size: 29MB
INFO: Current Start LSN: 0/D000028, TLI: 1
INFO: Start transferring data files
INFO: Data files are transferred, time elapsed: 0
INFO: wait for pg_stop_backup()
INFO: pg_stop backup() successfully executed
INFO: Wait for WAL segment /home/backups/backups/main/S1M89T/database/pg_wal/00000001000000000000000E to be streamed
INFO: stop_lsn: 0/E000000
INFO: Getting the Recovery Time from WAL
INFO: Failed to find Recovery Time in WAL, forced to trust current_timestamp
INFO: Syncing backup files to disk
INFO: Backup files are synced, time elapsed: 10s
INFO: Validating backup S1M89T
INFO: Backup S1M89T data files are valid
INFO: Backup S1M89T resident size: 61MB
INFO: Backup S1M89T completed
postgres@dba5-slave:~$ pg_probackup-15 show

BACKUP INSTANCE 'main'
================================================================================================================================
 Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI  Time  Data   WAL  Zratio  Start LSN  Stop LSN   Status
================================================================================================================================
 main      15       S1M89T  2023-09-26 23:08:22+00  FULL  STREAM    1/0   24s  29MB  32MB    1.00  0/D000028  0/E000028  OK
 main      15       S1M891  2023-09-26 23:07:54+00  FULL  STREAM    1/0   23s  29MB  32MB    1.00  0/C000028  0/D000028  OK

```
