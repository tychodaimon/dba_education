## Настройка дисков для Постгреса

### Cоздал виртуальную машину dba2-1 c Ubuntu 20.04 LTS (bionic) в ЯО 
#### Установил на нее PostgreSQL 15 
```
ssh dba2-1@158.160.45.254
dba2-1@dba2-1:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
dba2-1@dba2-1:~$ sudo wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
dba2-1@dba2-1:~$ sudo apt update
dba2-1@dba2-1:~$ sudo apt install postgresql-15

dba2-1@dba2-1:~$ pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file  
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

#### Сделал произвольную таблицу
```
dba2-1@dba2-1:~$ sudo su -
root@dba2-1:~# su - postgres
postgres@dba2-1:~$ psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=#
\q
postgres@dba2-1:~$
logout
root@dba2-1:~#
logout
dba2-1@dba2-1:~$
logout
Connection to 51.250.7.251 closed.
```
### Создал новый диск в ЯО и подключил его к виртуальной машине dba2-1
#### Создал на нем раздел datapartition с файловой системой ext4
##### Установил parted
```
root@dba2-1:~# sudo apt install parted
```
##### вывел ошибки разделов чтобы определить новый диск
```
root@dba2-1:~# parted -l | grep Error

Error: /dev/vdb: unrecognised disk label
```
##### вывел список устройств
```
root@dba2-1:~# lsblk

NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS  
loop0    7:0    0  63.3M  1 loop /snap/core20/1822  
loop1    7:1    0 111.9M  1 loop /snap/lxd/24322  
loop2    7:2    0  49.8M  1 loop /snap/snapd/18357  
vda    252:0    0    10G  0 disk  
├─vda1 252:1    0     1M  0 part  
└─vda2 252:2    0    10G  0 part /  
vdb    252:16   0    10G  0 disk
```

##### установил для /dev/vdb таблицу разделов GPT
```
root@dba2-1:~# sudo parted /dev/vdb mklabel gpt
```
##### задал основной раздел на диске
```
root@dba2-1:~# sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
```
##### отформатировал диск в ext4
```
root@dba2-1:~# mkfs.ext4 -L datapartition /dev/vdb1

mke2fs 1.46.5 (30-Dec-2021)  
Creating filesystem with 2620928 4k blocks and 655360 inodes  
Filesystem UUID: 0d81a682-2deb-4733-ac32-d82d8bdb9506  
Superblock backups stored on blocks:  
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632  
  
Allocating group tables: done  
Writing inode tables: done  
Creating journal (16384 blocks): done  
Writing superblocks and filesystem accounting information: done
``` 
##### Задал ему метку second
```
root@dba2-1:~# e2label /dev/vdb1 second
```
##### Убедился что система видит новый диск
```
root@dba2-1:~# lsblk --fs

NAME   FSTYPE   FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS  
loop0  squashfs 4.0                                                     0   100% /snap/core20/1822  
loop1  squashfs 4.0                                                     0   100% /snap/lxd/24322  
loop2  squashfs 4.0                                                     0   100% /snap/snapd/18357  
vda  
├─vda1  
└─vda2 ext4     1.0          ed465c6e-049a-41c6-8e0b-c8da348a3577      5G    44% /  
vdb  
└─vdb1 ext4     1.0   second 0d81a682-2deb-4733-ac32-d82d8bdb9506  
```

##### Добавил директорию для монтирования
```
root@dba2-1:~# mkdir -p /mnt/data
```
##### Попробовал замонтировать ее вручную
```
root@dba2-1:~# mount -o defaults /dev/vdb1 /mnt/data
```
##### Добавил запись для автомонтирования в fstab
```
root@dba2-1:~# sudo sh -c 'echo "UUID=0d81a682-2deb-4733-ac32-d82d8bdb9506 /mnt/data ext4 defaults 0 2" >> /etc/fstab'
root@dba2-1:~# cat /etc/fstab

#  /etc/fstab: static file system information.  
#   
#  Use 'blkid' to print the universally unique identifier for a  
#  device; this may be used with UUID= as a more robust way to name devices  
#  that works even if disks are added and removed. See fstab(5).  
#   
#  <file system> <mount point>   <type>  <options>       <dump>  <pass>  
#  / was on /dev/vda2 during curtin installation  
/dev/disk/by-uuid/ed465c6e-049a-41c6-8e0b-c8da348a3577 / ext4 defaults 0 1  
UUID=0d81a682-2deb-4733-ac32-d82d8bdb9506 /mnt/data ext4 defaults 0 2
```
##### Убедился что диск монтируется автоматом
```
root@dba2-1:~# umount /mnt/data
root@dba2-1:~# mount -a /mnt/data
root@dba2-1:~# df -h

Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.1M  391M   1% /run
/dev/vda2       9.8G  4.3G  5.1G  47% /
tmpfs           2.0G  1.1M  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           392M  4.0K  392M   1% /run/user/1000
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
```
##### Перезагрузил виртуалку
```
root@dba2-1:~# reboot
Connection to 84.201.129.94 closed by remote host.
Connection to 84.201.129.94 closed.
```

##### Дал права на директорию data пользователю postgres и перенес туда данные кластера
```
dba2-1@dba2-1:~$ sudo su -
root@dba2-1:~# chown -R postgres:postgres /mnt/data/
root@dba2-1:~# mv /var/lib/postgresql/15 /mnt/data
```
##### Убедился что попытка запуска приводит к ошибке
```
root@dba2-1:~# sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
##### В файле postgresql.conf поменял значение параметра data_directory
```
root@dba2-1:~# vi /etc/postgresql/15/main/postgresql.conf
```
>data_directory = '/var/lib/postgresql/15/main'  
на   
>data_directory = '/mnt/data/15/main'  
##### Кластер запустился успешно
```
root@dba2-1:~# sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Removed stale pid file.
```
##### Проверил тестовые данные
```
root@dba2-1:~# su - postgres
postgres@dba2-1:~$ psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=#
\q
postgres@dba2-1:~$
logout
root@dba2-1:~#
logout
dba2-1@dba2-1:~$
logout
Connection to 84.201.129.94 closed.
```

### Задание с *
#### Отправил запрос на отсоединение диска в ЯО на живую (не выключая виртуалку)
##### Проверил как отреагировала система
```
dba2-1@dba2-1:~$ ls /mnt/data/
ls: reading directory '/mnt/data/': Input/output error

dba2-1@dba2-1:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           392M  1.1M  391M   1% /run
/dev/vda2       9.8G  4.3G  5.1G  46% /
tmpfs           2.0G  1.1M  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/vdb1       9.8G   39M  9.2G   1% /mnt/data
tmpfs           392M  4.0K  392M   1% /run/user/1000

dba2-1@dba2-1:~$ sudo lsblk --fs
NAME   FSTYPE   FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                    0   100% /snap/core20/1822
loop1  squashfs 4.0                                                    0   100% /snap/lxd/24322
loop2  squashfs 4.0                                                    0   100% /snap/snapd/18357
vda
├─vda1
└─vda2 ext4     1.0         ed465c6e-049a-41c6-8e0b-c8da348a3577      5G    44% /
```

#### Создал еще одну виртуалку dba2-1 и подключил к ней этот диск в ЯО
##### Установил на нее Postgres 15
```
user@daemaken ~ % ssh dba2-2@158.160.45.254
dba2-2@dba2-2:~$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
dba2-2@dba2-2:~$ sudo wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
dba2-2@dba2-2:~$ sudo apt update
dba2-2@dba2-2:~$ sudo apt install postgresql-15
dba2-2@dba2-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
##### Удалил ее директорию с данными
```
dba2-2@dba2-2:~$ sudo rm -rf /var/lib/postgresql/15/main

dba2-2@dba2-2:~$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
15  main    5432 online <unknown> /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
##### Подключил диск к второй виртуалке
```
dba2-2@dba2-2:~$ lsblk --fs
NAME   FSTYPE   FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                     0   100% /snap/core20/1822
loop1  squashfs 4.0                                                     0   100% /snap/lxd/24322
loop2  squashfs 4.0                                                     0   100% /snap/snapd/18357
vda
├─vda1
└─vda2 ext4     1.0          ed465c6e-049a-41c6-8e0b-c8da348a3577    5.2G    42% /
vdb
└─vdb1 ext4     1.0   second 0d81a682-2deb-4733-ac32-d82d8bdb9506

dba2-2@dba2-2:~$ sudo mkdir -p /mnt/data
dba2-2@dba2-2:~$ sudo mount -o defaults /dev/vdb1 /mnt/data
dba2-2@dba2-2:~$ ls /mnt/data/
15  lost+found

dba2-2@dba2-2:~$ sudo sh -c 'echo "UUID=0d81a682-2deb-4733-ac32-d82d8bdb9506 /mnt/data ext4 defaults 0 2" >> /etc/fstab'
```
##### Поменял data_directory на этот диск
```
dba2-2@dba2-2:~$ sudo vi /etc/postgresql/15/main/postgresql.conf
```
>data_directory = '/var/lib/postgresql/15/main'  
>на  
>data_directory = '/mnt/data/15/main'  
##### Перезагрузил кластер для применения настроек
```
dba2-2@dba2-2:~$ sudo pg_ctlcluster 15 main restart
dba2-2@dba2-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```
##### Убедился что тестовые данные читаются
```
dba2-2@dba2-2:~$ sudo su -
root@dba2-2:~# su - postgres
postgres@dba2-2:~$ psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)

postgres=#
\q
postgres@dba2-2:~$
logout
root@dba2-2:~#
logout
dba2-2@dba2-2:~$
logout
Connection to 158.160.45.254 closed.
```
