# запускаю PostgreSQL сервис в ЯО

#### Ставлю клиента для облака
```
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
yc init
```

#### Поднимаю Managed Service for PostgreSQL
```
sudo apt update && sudo apt-get install jq

export NETWORK=$(yc vpc network get default --format json | jq -r '.id')
export FOLDER=$(yc config get folder-id)
export SUBNET=$(yc vpc subnet get default-ru-central1-a --format json | jq -r '.id')

yc postgresql cluster create \
 --folder-id $FOLDER \
 --name pg20 \
 --postgresql-version 16 \
 --environment production \
 --labels prod=postgres \
 --network-name default \
 --resource-preset m3-c2-m16 \
 --host zone-id=ru-central1-a,subnet-id=$SUBNET,assign-public-ip \
 --disk-type network-ssd \
 --disk-size 10 \
 --user name=otus,password=admin123$,conn-limit=50 \
 --database name=otus,owner=otus \
 --backup-window-start 01:00:00 \
 --backup-retain-period-days 7 \
 --websql-access \
 --serverless-access \
 --datalens-access \
 --datatransfer-access \
 --deletion-protection=false \
 --async
id: c9qf027l0lmnah19k1sv
description: Create PostgreSQL cluster
created_at: "2023-12-08T18:57:16.111550Z"
created_by: ajejoebv786o127le25i
modified_at: "2023-12-08T18:57:16.111550Z"
metadata:
  '@type': type.googleapis.com/yandex.cloud.mdb.postgresql.v1.CreateClusterMetadata
  cluster_id: c9qdsi7le2vju6agp8b8
```

#### Мониторю статус инициализации
```
watch "yc postgresql cluster get pg20 --format=json | jq -r '.status'"
Every 2.0s: yc postgresql cluster get pg20 --format=json | jq -r '.status'                                                                                                                                    dba20: Fri Dec  8 19:01:02 2023

CREATING

RUNNING
```

#### Проверяю настройки
```
yc postgresql cluster get pg20 --format=json
{
  "id": "c9qdsi7le2vju6agp8b8",
  "folder_id": "b1gmhmstvdpisspjlv8m",
  "created_at": "2023-12-08T18:57:16.111550Z",
  "name": "pg20",
  "labels": {
    "prod": "postgres"
  },
  "environment": "PRODUCTION",
  "monitoring": [
    {
      "name": "Console",
      "description": "Console charts",
      "link": "https://console.cloud.yandex.ru/folders/b1gmhmstvdpisspjlv8m/managed-postgresql/cluster/c9qdsi7le2vju6agp8b8?section=monitoring"
    }
  ],
  "config": {
    "version": "16",
    "postgresql_config_16": {
      "user_config": {}
    },
    "resources": {
      "resource_preset_id": "m3-c2-m16",
      "disk_size": "10737418240",
      "disk_type_id": "network-ssd"
    },
    "autofailover": true,
    "backup_window_start": {
      "hours": 1
    },
    "backup_retain_period_days": "7",
    "access": {
      "data_lens": true,
      "web_sql": true,
      "serverless": true,
      "data_transfer": true
    },
    "performance_diagnostics": {
      "sessions_sampling_interval": "60",
      "statements_sampling_interval": "600"
    },
    "disk_size_autoscaling": {}
  },
  "network_id": "enp0bg64k6tdf5uqk6li",
  "health": "ALIVE",
  "status": "RUNNING",
  "maintenance_window": {
    "anytime": {}
  }
}
```

#### Подключаюсь к кластеру
```
sudo apt install --yes postgresql-client

mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
    --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt

psql "host=rc1a-6wagrl5p8fae2jca.mdb.yandexcloud.net \
    port=6432 \
    sslmode=verify-full \
    dbname=otus \
    user=otus \
    target_session_attrs=read-write"
Password for user otus:
psql (14.10 (Ubuntu 14.10-0ubuntu0.22.04.1), server 16.0 (Ubuntu 16.0-201-yandex.56587.637d925cc2))
WARNING: psql major version 14, server major version 16.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

otus=> \dt
Did not find any relations.
otus=> \l
                             List of databases
   Name    |  Owner   | Encoding | Collate | Ctype |   Access privileges
-----------+----------+----------+---------+-------+-----------------------
 otus      | otus     | UTF8     | C       | C     | =T/otus              +
           |          |          |         |       | otus=CTc/otus        +
           |          |          |         |       | admin=c/otus         +
           |          |          |         |       | postgres=c/otus      +
           |          |          |         |       | monitor=c/otus       +
           |          |          |         |       | mdb_odyssey=c/otus
 postgres  | postgres | UTF8     | C       | C     |
 template0 | postgres | UTF8     | C       | C     | =c/postgres          +
           |          |          |         |       | postgres=CTc/postgres
 template1 | postgres | UTF8     | C       | C     | =c/postgres          +
           |          |          |         |       | postgres=CTc/postgres
(4 rows)

otus=>
```
