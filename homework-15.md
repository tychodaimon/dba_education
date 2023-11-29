# Развернуть параллельный кластер (Greenplum)

#### Подготавливаем 4 ноды
```
sudo -u gpadmin ssh-keygen -t rsa -b 4096 -q -f /home/gpadmin/.ssh/id_rsa -N ''
vi greenplum.sh
cat ./greenplum.sh


#! /bin/bash

# Check we are a root
if [ "$EUID" -ne 0 ]
    then echo "Please run this script as root"
    exit
fi

REPO="/etc/apt/sources.list.d/greenplum-ubuntu-db-bionic.list"
PIN="/etc/apt/preferences.d/99-greenplum"

echo "Add required repositories"
touch $REPO
cat > $REPO <<REPOS
deb http://ppa.launchpad.net/greenplum/db/ubuntu bionic main
deb http://ru.archive.ubuntu.com/ubuntu bionic main
REPOS

echo "Configure repositories"
touch $PIN
cat > $PIN <<PIN_REPO
Package: *
Pin: release v=18.04
Pin-Priority: 1
PIN_REPO

echo "Repositories described in \$REPO"
echo "Repositories configuration in \$PIN"
echo "Installing greenplum"
gpg --keyserver keyserver.ubuntu.com --recv 3C6FDC0C01D86213
gpg --export --armor 3C6FDC0C01D86213 | sudo apt-key add -
apt update && apt install greenplum-db-6 -y



sudo chmod +x ./greenplum.sh
sudo ./greenplum.sh
dpkg-query -l | grep greenplum-db
ii  greenplum-db-6                         6.26.0-1                                        amd64        Greenplum Database
sudo chown -R gpadmin:gpadmin /opt/greenplum-db-6.26.0
echo "source /opt/greenplum-db-6.26.0/greenplum_path.sh" > ~/.bashrc
sudo su gpadmin
which gpssh
/opt/greenplum-db-6.26.0/bin/gpssh
cat ~/.ssh/id_rsa.pub
```

#### Заливаем собранные публичные ключи со всех ноды в каждую ноду и прописываем хосты
```
vi ~/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCnRa7LwxgVG7c4gEgAFTpnM81Lq/S7QdCxeK5yPjW9vu9M6LUWJtItUZ3T+EIHfNPRMioSD+A/S+TtPYSjr4Sb/We2ciAwhOCNj1f7BDPC4kURj2xRuJptdvy4JXiT5Nl78jzWt7IIgyV5j6C+E98T4F2ypuLkW7a8+78kEv+d7FQ0BkfE6O+VdOySGdhuK3rw7KJGME3pdOeto4HlSZizmVndwk5e9AyCZvY8HR2VZMkBjWTaq2eRFYqhZC8pQWY5sF/gg50YQXdCZOXQUszeM0rJFeTRxocqAEvDDCr5A1SGNk9+xgHsRVTZwls6vISyO1oZ+xbMroFgD/QQ+IETErvUK1w0GMsqBZJtxYuj3NYCzrOGSq/GZMshJR19wT52K/cxHNu4gD/073UHZdAYIFneC9ty3Ad2bYqNtWOxsnCTNOEjFDM1/gUHmDIVlQw9o4XQOA4THCdE9EnHxCFBehALBgcOhBuf5RC8dGtgUSq9TZ/afsck2nkRg7ZULcspfKzPsLrIDY9zj2vpSn3Xi6gjO5bcCgo82Px+iObgBTrY/ervEBHynh9/f/C5y0K/47N722CPTY3UJNhxhrnH1Ox5q9kmkDUOw12STqX/JQf1tJhirdUkS3cNp/0cDnH0s7lv8wJQEVAjujq+3uUIZaoVTcypla9VQOot/9eavQ== gpadmin@gp1
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC2d6hUlr7AfS5f8DAzbN+ay5J0bAaoJ0BlfIn1XBodytET95jGFN5gXhuNb+uY7brwRvTcAQ7CRk6b3/qBKo/kJP3OPT3jz6uvhyb4dgIb0ODMRacWqaf3XVp2qO0BHADQUMdW8E1Aczo4TRz4iLHgbPQ19ycKvO0tif/sCmVs05IkrVp27oc28kdcLjsf34fHC8isYUAQNaPN6hvhAB41nPsIEOGdtpxMQWu8mFGcf8wdf2ek7K5BAYxT0kPgZIazTLp5mfyeEf55Dg7matOG/oF3eIqmvagQQCBb75Xb84joOa2I7kzTguhGrRH/FYcLjYg/PSOddvN4Y2sLcVoD7tTnCIgeRQXq9GqxKjery5UUGxQ+Ie5jwNDxXEy+UvCd5ugG2nI2fygSBlHJ+DXtTo2NNvjEnwGUrP9W9/bKh0fR2Qe/9laW+8bqSeyWy7OIGXBfQWxZbxXTxztCbxe6PNgznDrhcZskKi4tKggFOs5FcCeZN9oPwxGqkeTG/P9PoNa3E1eLLSuie8mBTFvx6joUgJz++VWTvW4Z6bUFkziPwD+VM3otcBvFUj7eujhusxDWic1Zefr70wno/VHHKQbP/JlLM42US6OdyyaFymjjAZdTo7nda55kx4aZW017SEHkAa+Yg8ynwLBtQYNqgfOLHIVrx6GSAvAkI37Pzw== gpadmin@gp2
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDV1MldnHuEaxmCTlgjDeicYVZp6I0rpbOsWMHGIOtNb4zAJvM7BaVTzjART/rbk1VqfPaKzt4TMUIRGFykh/Ep8fJeV6JCQyFQ6hudJW81LH/bwG2oZ+2SZ5pQCWsP/7aonjoAGTYYYGeOc2RELvKejckvL2zZQ6E5PraEyhsSERCk0z3b2iw2tv5v+rstOm0TJABg9DZ15g1YMdUvsMAmhy6DkbhavHGNlJq1YW500ZaYIOb3n7vmjfUWT1nrDT9yCt7HPDM0+Bd4UEIIeJi3jbiECtuiDTdLboOs4WXmpdRqhO7X9dW/Ld4w37u0zUAwbr130UnOIEUv0K3DSgWYIxeKadaGmuJIhLdMKAaUYqYxn84Eyq2+DHXJkLv3fyrE//5QCMugHpvQBloubhPX8I3LKiavQHZqyJdOvl9cn/h9B8/xzkJpmQqsC7YLAeCEkF2zqeTpZrot6KIPRU3356pNNI+J0iq1Sa3k7BRZOrpmEgm7UvAVqRatAFtsb9fFeFhpejvIyzutv7VsKlSLNYCdoZ9uzZUJ1HRUP/LUrqpOPhtc08X4dx/BHYxSHQTH7ImHDLLT5cW6X+xcmDB1htUsKNBW2SsEerYiQ5JmAJVlMFuyC1lzB0+1oRc4Ye/Mb3cLVs5Kl2OQpwYwGPpaigBQDfg+jFD7eL/iA34Yuw== gpadmin@gp3
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDhqFUjBscFAl4bj/kuPU19T09SsGH6TNjxKy7LMKuB3MZleb2XSjt6Ns/9/CT2us2pvC233ASQJznA+1vPvr0HRGwjJbrSU1s09aBY9kUU4extzXKk/oj0XW94+CT8TSYx714dEJT9IXqmXCJtDNCgzDuKS0UHRojnv8OtV243yKQB/nP91KRPw6RuBfECuPrODJQW6ocKHDJ2VGKnwOKyuN6hsT2sfprzgu/AVxFfnVaCV5qUMiN0NW0Jm2dJlpjUBgXnvrLx9jghSjmUdVHMjAQbFE9k5mNDbbd9BJ0mtUjXxOhy2wAShupj9zX6YC6lKqkFJYWUOIDfyLCYZFc/+YyQQfzEpo23l1fsYTWw46HpSy2TVbPhds0Y/Bov0I5ohGPHV9R2fNn7RcMer3/kJTybCL+c5TxClqwUevfyCpgnFZfUjAv5Lujjd7sn1ra2wP2U9RjDoWX0yjJ29ko+ZWuSuJMEu/hfJt4Hjor2syUPwxxmp2xK6yfGkjYnkxRzOH9dE0fxXu/mIciyKhMTQDDJdhbQNxAptD/JCnMRV8z+WQq4SCgqR2lfM42D+KZPVMh84x5ZhnUHM6moPKhKaI9+Q5NPn24bfWqxdXrmTq+20I2GtbkWwbUG0lmlSAR5UHhkpXYl3ta0v5uGYLP+uRPsH59dqNoKPbH5+Yn24w== gpadmin@gp4

vi ~/hostfile_exkeys
cat ~/hostfile_exkeys
gp1
gp2
gp3
gp4
```

#### Инициализируем обмен ключами
```
gpssh-exkeys -f hostfile_exkeys
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] retrieving credentials from remote hosts
  ... send to gp2
  ... send to gp3
  ... send to gp4

[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with gp2
  ... finished key exchange with gp3
  ... finished key exchange with gp4

[INFO] completed successfully



gpssh -f hostfile_exkeys -e 'ls -l /opt/greenplum-db-6.26.0'
[gp2] ls -l /opt/greenplum-db-6.26.0
[gp2] total 40
[gp2] drwxr-xr-x 8 gpadmin gpadmin 4096 Nov 28 22:18 bin
[gp2] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:18 docs
[gp2] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:18 etc
[gp2] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:18 ext
[gp2] -rw-r--r-- 1 gpadmin gpadmin 1123 Nov 16 03:54 greenplum_path.sh
[gp2] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:18 include
[gp2] drwxr-xr-x 5 gpadmin gpadmin 4096 Nov 28 22:18 lib
[gp2] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:18 libexec
[gp2] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:18 sbin
[gp2] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:18 share
[gp1] ls -l /opt/greenplum-db-6.26.0
[gp1] total 40
[gp1] drwxr-xr-x 8 gpadmin gpadmin 4096 Nov 28 22:08 bin
[gp1] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:07 docs
[gp1] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:07 etc
[gp1] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:07 ext
[gp1] -rw-r--r-- 1 gpadmin gpadmin 1123 Nov 16 03:54 greenplum_path.sh
[gp1] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:07 include
[gp1] drwxr-xr-x 5 gpadmin gpadmin 4096 Nov 28 22:07 lib
[gp1] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:07 libexec
[gp1] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:08 sbin
[gp1] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:07 share
[gp4] ls -l /opt/greenplum-db-6.26.0
[gp4] total 40
[gp4] drwxr-xr-x 8 gpadmin gpadmin 4096 Nov 28 22:26 bin
[gp4] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:26 docs
[gp4] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:26 etc
[gp4] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:26 ext
[gp4] -rw-r--r-- 1 gpadmin gpadmin 1123 Nov 16 03:54 greenplum_path.sh
[gp4] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:26 include
[gp4] drwxr-xr-x 5 gpadmin gpadmin 4096 Nov 28 22:26 lib
[gp4] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:26 libexec
[gp4] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:26 sbin
[gp4] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:26 share
[gp3] ls -l /opt/greenplum-db-6.26.0
[gp3] total 40
[gp3] drwxr-xr-x 8 gpadmin gpadmin 4096 Nov 28 22:24 bin
[gp3] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:24 docs
[gp3] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:24 etc
[gp3] drwxr-xr-x 3 gpadmin gpadmin 4096 Nov 28 22:24 ext
[gp3] -rw-r--r-- 1 gpadmin gpadmin 1123 Nov 16 03:54 greenplum_path.sh
[gp3] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:24 include
[gp3] drwxr-xr-x 5 gpadmin gpadmin 4096 Nov 28 22:24 lib
[gp3] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:24 libexec
[gp3] drwxr-xr-x 2 gpadmin gpadmin 4096 Nov 28 22:24 sbin
[gp3] drwxr-xr-x 4 gpadmin gpadmin 4096 Nov 28 22:24 share
```

#### Создаем рабочие директории на мастере/реплике и на нодах
```
# on gp1 (master)
sudo mkdir -p /data/master
sudo chown gpadmin:gpadmin /data/master

# on gp2 (standby)
sudo mkdir -p /data/master
sudo chown gpadmin:gpadmin /data/master

# on gp3-4 (nodes)
sudo mkdir -p /data/primary
sudo mkdir -p /data/mirror
sudo chown -R gpadmin /data/*
```

#### На всех машинах прописываем ноды
```
vi ~/hostfile_gpinitsystem
cat ~/hostfile_gpinitsystem
gp3
gp4
```

#### На мастере создаем и редактируем конфиг
```
gpadmin@gp1:~$ mkdir /home/gpadmin/gpconfigs
gpadmin@gp1:~$ cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config /home/gpadmin/gpconfigs/gpinitsystem_config
gpadmin@gp1:~$ vi /home/gpadmin/gpconfigs/gpinitsystem_config
gpadmin@gp1:~$ cat /home/gpadmin/gpconfigs/gpinitsystem_config


# FILE NAME: gpinitsystem_config

# Configuration file needed by the gpinitsystem

################################################
#### REQUIRED PARAMETERS
################################################

#### Name of this Greenplum system enclosed in quotes.
ARRAY_NAME="Greenplum Data Platform"

#### Naming convention for utility-generated data directories.
SEG_PREFIX=gpseg

#### Base number by which primary segment port numbers
#### are calculated.
PORT_BASE=6000

#### File system location(s) where primary segment data directories
#### will be created. The number of locations in the list dictate
#### the number of primary segments that will get created per
#### physical host (if multiple addresses for a host are listed in
#### the hostfile, the number of segments will be spread evenly across
#### the specified interface addresses).
declare -a DATA_DIRECTORY=(/data/primary)

#### OS-configured hostname or IP address of the master host.
MASTER_HOSTNAME=gp1

#### File system location where the master data directory
#### will be created.
MASTER_DIRECTORY=/data/master

#### Port number for the master instance.
MASTER_PORT=5432

#### Shell utility used to connect to remote hosts.
TRUSTED_SHELL=ssh

#### Maximum log file segments between automatic WAL checkpoints.
CHECK_POINT_SEGMENTS=8

#### Default server-side character set encoding.
ENCODING=UNICODE

################################################
#### OPTIONAL MIRROR PARAMETERS
################################################

#### Base number by which mirror segment port numbers
#### are calculated.
MIRROR_PORT_BASE=7000

#### File system location(s) where mirror segment data directories
#### will be created. The number of mirror locations must equal the
#### number of primary locations as specified in the
#### DATA_DIRECTORY parameter.
declare -a MIRROR_DATA_DIRECTORY=(/data/mirror)


################################################
#### OTHER OPTIONAL PARAMETERS
################################################

#### Create a database of this name after initialization.
DATABASE_NAME=otus

#### Specify the location of the host address file here instead of
#### with the -h option of gpinitsystem.
MACHINE_LIST_FILE=/home/gpadmin/hostfile_gpinitsystem
```

#### Дополнительно увеличил лимиты операционки
```
sudo vi /etc/security/limits.conf
cat /etc/security/limits.conf

* soft nproc 65535
* hard nproc 65535
* soft nofile 65535
* hard nofile 65535

linuxhint soft nproc 100000
linuxhint hard nproc 100000
linuxhint soft nofile 100000
linuxhint hard nofile 100000
```

#### Инициализируем кластер
```
gpadmin@gp1:~$ gpinitsystem -c ./gpconfigs/gpinitsystem_config -h ./hostfile_gpinitsystem -s gp2 --mirror-mode=spread
20231129:00:43:36:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20231129:00:43:36:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Reading Greenplum configuration file ./gpconfigs/gpinitsystem_config
20231129:00:43:36:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Locale has not been set in ./gpconfigs/gpinitsystem_config, will set to default value
20231129:00:43:36:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Locale set to en_US.utf8
20231129:00:43:36:000797 gpinitsystem:gp1:gpadmin-[INFO]:-MASTER_MAX_CONNECT not set, will set to default value 250
20231129:00:43:37:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Checking configuration parameters, Completed
20231129:00:43:37:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
..
20231129:00:43:37:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Configuring build for standard array
20231129:00:43:37:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Sufficient hosts for spread mirroring request
20231129:00:43:37:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20231129:00:43:37:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Building primary segment instance array, please wait...
..
20231129:00:43:38:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Building spread mirror array type , please wait...
..
20231129:00:43:39:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Checking Master host
20231129:00:43:39:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Checking new segment hosts, please wait...
....
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Checking new segment hosts, Completed
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Greenplum Database Creation Parameters
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:---------------------------------------
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master Configuration
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:---------------------------------------
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master instance name       = Greenplum Data Platform
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master hostname            = gp1
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master port                = 5432
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master instance dir        = /data/master/gpseg-1
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master LOCALE              = en_US.utf8
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Greenplum segment prefix   = gpseg
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master Database            = otus
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master connections         = 250
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master buffers             = 128000kB
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Segment connections        = 750
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Segment buffers            = 128000kB
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Checkpoint segments        = 8
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Encoding                   = UNICODE
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Postgres param file        = Off
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Initdb to be used          = /opt/greenplum-db-6.26.0/bin/initdb
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-GP_LIBRARY_PATH is         = /opt/greenplum-db-6.26.0/lib
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-HEAP_CHECKSUM is           = on
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-HBA_HOSTNAMES is           = 0
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Ulimit check               = Passed
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Array host connect type    = Single hostname per node
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master IP address [1]      = ::1
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master IP address [2]      = 10.128.0.17
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Master IP address [3]      = fe80::d20d:10ff:fe11:863a
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Standby Master             = gp2
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Number of primary segments = 1
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Standby IP address         = ::1
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Standby IP address         = 10.128.0.35
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Standby IP address         = fe80::d20d:12ff:fe90:2cb4
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total Database segments    = 2
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Trusted shell              = ssh
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Number segment hosts       = 2
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Mirror port base           = 7000
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Number of mirror segments  = 1
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Mirroring config           = ON
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Mirroring type             = Spread
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:----------------------------------------
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Greenplum Primary Segment Configuration
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:----------------------------------------
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-gp3 	6000 	gp3 	/data/primary/gpseg0 	2
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-gp4 	6000 	gp4 	/data/primary/gpseg1 	3
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:---------------------------------------
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Greenplum Mirror Segment Configuration
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:---------------------------------------
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-gp4 	7000 	gp4 	/data/mirror/gpseg0 	4
20231129:00:43:46:000797 gpinitsystem:gp1:gpadmin-[INFO]:-gp3 	7000 	gp3 	/data/mirror/gpseg1 	5

Continue with Greenplum creation Yy|Nn (default=N):Y

20231129:00:44:14:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Building the Master instance database, please wait...
20231129:00:44:19:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Starting the Master in admin mode
20231129:00:44:19:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Commencing parallel build of primary segment instances
20231129:00:44:19:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
..
20231129:00:44:19:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
...............
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:------------------------------------------------
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Parallel process exit status
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:------------------------------------------------
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total processes marked as completed           = 2
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total processes marked as killed              = 0
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total processes marked as failed              = 0
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:------------------------------------------------
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Removing back out file
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-No errors generated from parallel processes
20231129:00:44:34:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Restarting the Greenplum instance in production mode
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Starting gpstop with args: -a -l /home/gpadmin/gpAdminLogs -m -d /data/master/gpseg-1
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Gathering information and validating the environment...
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Obtaining Segment details from master...
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Greenplum Version: 'postgres (Greenplum Database) 6.26.0 build commit:99181e5ec826aa5e7d18926dce2f24a061ff8183 Open Source'
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Commencing Master instance shutdown with mode='smart'
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Master segment instance directory=/data/master/gpseg-1
20231129:00:44:35:003666 gpstop:gp1:gpadmin-[INFO]:-Stopping master segment and waiting for user connections to finish ...
server shutting down
20231129:00:44:36:003666 gpstop:gp1:gpadmin-[INFO]:-Attempting forceful termination of any leftover master process
20231129:00:44:36:003666 gpstop:gp1:gpadmin-[INFO]:-Terminating processes for segment /data/master/gpseg-1
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Starting gpstart with args: -a -l /home/gpadmin/gpAdminLogs -d /data/master/gpseg-1
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Gathering information and validating the environment...
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Greenplum Binary Version: 'postgres (Greenplum Database) 6.26.0 build commit:99181e5ec826aa5e7d18926dce2f24a061ff8183 Open Source'
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Greenplum Catalog Version: '301908232'
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Starting Master instance in admin mode
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Obtaining Segment details from master...
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Setting new master era
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Master Started...
20231129:00:44:38:003984 gpstart:gp1:gpadmin-[INFO]:-Shutting down master
20231129:00:44:41:003984 gpstart:gp1:gpadmin-[INFO]:-Commencing parallel segment instance startup, please wait...
.
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-Process results...
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-   Successful segment starts                                            = 2
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-   Failed segment starts                                                = 0
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-   Skipped segment starts (segments are marked down in configuration)   = 0
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-Successfully started 2 of 2 segment instances
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-Starting Master instance gp1 directory /data/master/gpseg-1
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-Command pg_ctl reports Master gp1 instance active
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-Connecting to dbname='template1' connect_timeout=15
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-No standby master configured.  skipping...
20231129:00:44:42:003984 gpstart:gp1:gpadmin-[INFO]:-Database successfully started
20231129:00:44:42:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Completed restart of Greenplum instance in production mode
20231129:00:44:43:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Commencing parallel build of mirror segment instances
20231129:00:44:43:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
..
20231129:00:44:43:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
........
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:------------------------------------------------
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Parallel process exit status
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:------------------------------------------------
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total processes marked as completed           = 2
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total processes marked as killed              = 0
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Total processes marked as failed              = 0
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:------------------------------------------------
20231129:00:44:51:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Starting initialization of standby master gp2
20231129:00:44:51:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Validating environment and parameters for standby initialization...
20231129:00:44:51:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Checking for data directory /data/master/gpseg-1 on gp2
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:------------------------------------------------------
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum standby master initialization parameters
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:------------------------------------------------------
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum master hostname               = gp1
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum master data directory         = /data/master/gpseg-1
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum master port                   = 5432
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum standby master hostname       = gp2
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum standby master port           = 5432
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum standby master data directory = /data/master/gpseg-1
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Greenplum update system catalog         = On
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Syncing Greenplum Database extensions to standby
20231129:00:44:52:004963 gpinitstandby:gp1:gpadmin-[INFO]:-The packages on gp2 are consistent.
20231129:00:44:53:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Adding standby master to catalog...
20231129:00:44:53:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Database catalog updated successfully.
20231129:00:44:53:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Updating pg_hba.conf file...
20231129:00:44:54:004963 gpinitstandby:gp1:gpadmin-[INFO]:-pg_hba.conf files updated successfully.
20231129:00:44:57:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Starting standby master
20231129:00:44:57:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Checking if standby master is running on host: gp2  in directory: /data/master/gpseg-1
20231129:00:44:59:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Cleaning up pg_hba.conf backup files...
20231129:00:44:59:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Backup files of pg_hba.conf cleaned up successfully.
20231129:00:44:59:004963 gpinitstandby:gp1:gpadmin-[INFO]:-Successfully created standby master on gp2
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Successfully completed standby master initialization
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Scanning utility log file for any warning messages
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[WARN]:-*******************************************************
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[WARN]:-Scan of log file indicates that some warnings or errors
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[WARN]:-were generated during the array creation
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Please review contents of log file
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-/home/gpadmin/gpAdminLogs/gpinitsystem_20231129.log
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-To determine level of criticality
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-These messages could be from a previous run of the utility
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-that was called today!
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[WARN]:-*******************************************************
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Greenplum Database instance successfully created
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-------------------------------------------------------
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-To complete the environment configuration, please
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-update gpadmin .bashrc file with the following
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-1. Ensure that the greenplum_path.sh file is sourced
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-2. Add "export MASTER_DATA_DIRECTORY=/data/master/gpseg-1"
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-   to access the Greenplum scripts for this instance:
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-   or, use -d /data/master/gpseg-1 option for the Greenplum scripts
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-   Example gpstate -d /data/master/gpseg-1
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Script log file = /home/gpadmin/gpAdminLogs/gpinitsystem_20231129.log
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-To remove instance, run gpdeletesystem utility
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Standby Master gp2 has been configured
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-To activate the Standby Master Segment in the event of Master
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-failure review options for gpactivatestandby
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-------------------------------------------------------
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-The Master /data/master/gpseg-1/pg_hba.conf post gpinitsystem
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-has been configured to allow all hosts within this new
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-array to intercommunicate. Any hosts external to this
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-new array must be explicitly added to this file
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-Refer to the Greenplum Admin support guide which is
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-located in the /opt/greenplum-db-6.26.0/docs directory
20231129:00:44:59:000797 gpinitsystem:gp1:gpadmin-[INFO]:-------------------------------------------------------
```

#### Прописываем директорию с данными в переменные среды и проверяем состояние
```
gpadmin@gp1:~$ echo "export MASTER_DATA_DIRECTORY=/data/master/gpseg-1" >> ~/.bashrc
gpadmin@gp1:~$ sudo su gpadmin
gpadmin@gp1:~$ gpstate
20231129:00:50:06:005269 gpstate:gp1:gpadmin-[INFO]:-Starting gpstate with args:
20231129:00:50:06:005269 gpstate:gp1:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 6.26.0 build commit:99181e5ec826aa5e7d18926dce2f24a061ff8183 Open Source'
20231129:00:50:06:005269 gpstate:gp1:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 9.4.26 (Greenplum Database 6.26.0 build commit:99181e5ec826aa5e7d18926dce2f24a061ff8183 Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0, 64-bit compiled on Nov 16 2023 03:52:41'
20231129:00:50:06:005269 gpstate:gp1:gpadmin-[INFO]:-Obtaining Segment details from master...
20231129:00:50:06:005269 gpstate:gp1:gpadmin-[INFO]:-Gathering data from segments...
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-Greenplum instance status summary
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Master instance                                           = Active
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Master standby                                            = gp2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Standby master state                                      = Standby host passive
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total segment instance count from metadata                = 4
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Primary Segment Status
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total primary segments                                    = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total primary segment valid (at master)                   = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total primary segment failures (at master)                = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files missing              = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files found                = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs found                 = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files missing                   = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files found                     = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes missing                 = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes found                   = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Mirror Segment Status
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total mirror segments                                     = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total mirror segment valid (at master)                    = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total mirror segment failures (at master)                 = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files missing              = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid files found                = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs found                 = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files missing                   = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number of /tmp lock files found                     = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes missing                 = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number postmaster processes found                   = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number mirror segments acting as primary segments   = 0
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-   Total number mirror segments acting as mirror segments    = 2
20231129:00:50:07:005269 gpstate:gp1:gpadmin-[INFO]:-----------------------------------------------------
```
