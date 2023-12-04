# запустить HA и multi master PostgreSQL кластер в Kubernetes

#### Ставлю клиента для кубов и подключаю кластер
```
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
yc init
yc managed-kubernetes cluster get-credentials --id catemavr5tvc7tkfh9ko --external
kubectl delete all,ing,secrets,pvc,pv --all
kubectl get pods
```

#### Создаю secrets.yaml
```
echo -n 'dba16password' | base64
ZGJhMTZwYXNzd29yZA==

vi secrets.yaml
dba16@dba16gke:~/$ cat secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: citus-secrets
type: Opaque
data:
  password: ZGJhMTZwYXNzd29yZA==
```

#### Создаю entrypoint.yaml (помогает подключить worker-ноды по hostname)
```
vi ./entrypoint.yaml
dba16@dba16:~/$ cat ./entrypoint.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: entrypoint
data:
  entrypoint.sh: |-
    #!/bin/bash
    until psql --host=citus-master --username=postgres --command="SELECT * from master_add_node('${HOSTNAME}.citus-workers', 5432);"; do sleep 1; done &
    exec /usr/local/bin/docker-entrypoint.sh "$@"
```

#### Записываю master.yaml
```
vi master.yaml
dba16@dba16:~/$ cat ./master.yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: citus-master-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: yc-network-ssd
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: citus-master
  labels:
    app: citus-master
spec:
  selector:
    app: citus-master
  type: LoadBalancer
  ports:
   - port: 5432
     targetPort: 5432
     protocol: TCP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-master
spec:
  selector:
    matchLabels:
      app: citus-master
  serviceName: citus-master
  replicas: 1
  template:
    metadata:
      labels:
        app: citus-master
    spec:
      containers:
      - name: citus
        image: citusdata/citus:latest
        ports:
        - containerPort: 5432
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        volumeMounts:
        - name: storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - ./pg_healthcheck
          initialDelaySeconds: 60
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: citus-master-pvc
```

#### Записываю workers.yaml
```
vi ./workers.yaml
dba16@dba16gke:~/$ cat ./workers.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: citus-workers
  labels:
    app: citus-workers
spec:
  selector:
    app: citus-workers
  clusterIP: None
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-worker
spec:
  selector:
    matchLabels:
      app: citus-workers
  serviceName: citus-workers
  replicas: 3
  template:
    metadata:
      labels:
        app: citus-workers
    spec:
      containers:
      - name: citus-worker
        image: citusdata/citus:latest
        command: ["/entrypoint.sh"]  # Новый entrypoint
        args: ["postgres"]           # обязательно cmd
        ports:
        - containerPort: 5432
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: storage
          mountPath: /var/lib/postgresql/data
        - name: entrypoint           # Монтируем volume с entrypoint в корень контейнера
          mountPath: /entrypoint.sh
          subPath: entrypoint.sh
        livenessProbe:
          exec:
            command:
            - ./pg_healthcheck
          initialDelaySeconds: 60
      volumes:
        - name: entrypoint           # volume из configMap с entrypoint
          configMap:
            name: entrypoint
            defaultMode: 0775
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: yc-network-ssd
      resources:
        requests:
          storage: 10Gi
```

#### Применяю конфигурацию
```
kubectl apply -f ./secrets.yaml
kubectl apply -f ./entrypoint.yaml
kubectl apply -f ./master.yaml
kubectl apply -f ./workers.yaml

kubectl get pod -o wide
kubectl get nodes
kubectl get all

NAME                 READY   STATUS    RESTARTS   AGE
pod/citus-master-0   1/1     Running   0          8m36s
pod/citus-worker-0   1/1     Running   0          7m8s
pod/citus-worker-1   1/1     Running   0          6m41s
pod/citus-worker-2   1/1     Running   0          6m14s

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
service/citus-master    LoadBalancer   10.96.181.166   158.160.130.204   5432:31708/TCP   8m36s
service/citus-workers   ClusterIP      None            <none>            5432/TCP         7m9s
service/kubernetes      ClusterIP      10.96.128.1     <none>            443/TCP          13m

NAME                            READY   AGE
statefulset.apps/citus-master   1/1     8m36s
statefulset.apps/citus-worker   3/3     7m8s
```

#### Подключаюсь к citus через LoadBalancer
```

kubectl get secret citus-secrets -o yaml
apiVersion: v1
data:
  password: ZGJhMTZwYXNzd29yZA==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"ZGJhMTZwYXNzd29yZA=="},"kind":"Secret","metadata":{"annotations":{},"name":"citus-secrets","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2023-12-04T00:10:46Z"
  name: citus-secrets
  namespace: default
  resourceVersion: "334869"
  uid: e385eebd-a0a0-46c6-b5bf-57581c89702a
type: Opaque


echo 'ZGJhMTZwYXNzd29yZA==' | base64 -d
dba16password


psql -h 158.160.130.204 -U postgres -d postgres
Password for user postgres:
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1), server 16.1 (Debian 16.1-1.pgdg120+1))
WARNING: psql major version 14, server major version 16.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# CREATE EXTENSION citus;
ERROR:  extension "citus" already exists

postgres=# SELECT * FROM master_get_active_worker_nodes();
          node_name           | node_port
------------------------------+-----------
 citus-worker-2.citus-workers |      5432
 citus-worker-0.citus-workers |      5432
 citus-worker-1.citus-workers |      5432
(3 rows)
\q
```

#### Заливаю тестовые даные 
```
kubectl exec -it pod/citus-master-0 bash

root@citus-master-0:/# apt-get update
Get:1 http://apt.postgresql.org/pub/repos/apt bookworm-pgdg InRelease [123 kB]
Get:2 http://deb.debian.org/debian bookworm InRelease [151 kB]
Get:3 http://deb.debian.org/debian bookworm-updates InRelease [52.1 kB]
Get:4 http://apt.postgresql.org/pub/repos/apt bookworm-pgdg/main amd64 Packages [301 kB]
Get:5 http://deb.debian.org/debian-security bookworm-security InRelease [48.0 kB]
Get:6 http://deb.debian.org/debian bookworm/main amd64 Packages [8,780 kB]
Get:7 http://deb.debian.org/debian bookworm-updates/main amd64 Packages [6,668 B]
Get:8 http://deb.debian.org/debian-security bookworm-security/main amd64 Packages [106 kB]
Get:9 https://repos.citusdata.com/community/debian bookworm InRelease [28.0 kB]
Get:10 https://repos.citusdata.com/community/debian bookworm/main amd64 Packages [28.4 kB]
Fetched 9,625 kB in 3s (3,434 kB/s)
Reading package lists... Done

root@citus-master-0:/# apt-get install wget
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  wget
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 984 kB of archives.
After this operation, 3,692 kB of additional disk space will be used.
Get:1 http://deb.debian.org/debian bookworm/main amd64 wget amd64 1.21.3-1+b2 [984 kB]
Fetched 984 kB in 0s (2,862 kB/s)
debconf: delaying package configuration, since apt-utils is not installed
Selecting previously unselected package wget.
(Reading database ... 13038 files and directories currently installed.)
Preparing to unpack .../wget_1.21.3-1+b2_amd64.deb ...
Unpacking wget (1.21.3-1+b2) ...
Setting up wget (1.21.3-1+b2) ..

root@citus-master-0:/# mkdir /home/1
root@citus-master-0:/# chmod 777 /home/1
root@citus-master-0:/# cd /home/1
root@citus-master-0:/home/1# wget https://storage.googleapis.com/postgres13/1000000SalesRecords.csv
--2023-12-04 00:32:43--  https://storage.googleapis.com/postgres13/1000000SalesRecords.csv
Resolving storage.googleapis.com (storage.googleapis.com)... 173.194.222.207, 142.251.1.207, 64.233.165.207, ...
Connecting to storage.googleapis.com (storage.googleapis.com)|173.194.222.207|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 124793263 (119M) [application/vnd.ms-excel]
Saving to: ‘1000000SalesRecords.csv’

1000000SalesRecords.csv                                     100%[========================================================================================================================================>] 119.01M  26.7MB/s    in 5.1s

2023-12-04 00:32:50 (23.2 MB/s) - ‘1000000SalesRecords.csv’ saved [124793263/124793263]


root@citus-master-0:/home/1# psql -U postgres
psql (16.1 (Debian 16.1-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE TABLE test (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    Region VARCHAR(50),
    Country VARCHAR(50),
    ItemType VARCHAR(50),
    SalesChannel VARCHAR(20),
    OrderPriority VARCHAR(10),
    OrderDate VARCHAR(10),
    OrderID int,
    ShipDate VARCHAR(10),
    UnitsSold int,
    UnitPrice decimal(12,2),
    UnitCost decimal(12,2),
    TotalRevenue decimal(12,2),
    TotalCost decimal(12,2),
    TotalProfit decimal(12,2)
);
CREATE TABLE

postgres=# SELECT create_distributed_table('test', 'id');
 create_distributed_table
--------------------------

(1 row)

postgres=# copy test (Region,Country,ItemType,SalesChannel,OrderPriority,OrderDate,OrderID,ShipDate,UnitsSold,UnitPrice,UnitCost,TotalRevenue,TotalCost,TotalProfit) FROM '/home/1/1000000SalesRecords.csv' DELIMITER ',' CSV HEADER;
COPY 1000000

postgres=# SELECT rebalance_table_shards('test');
 rebalance_table_shards
------------------------

(1 row)

postgres=# SELECT * FROM pg_dist_shard;
 logicalrelid | shardid | shardstorage | shardminvalue | shardmaxvalue
--------------+---------+--------------+---------------+---------------
 test         |  102008 | t            | -2147483648   | -2013265921
 test         |  102009 | t            | -2013265920   | -1879048193
 test         |  102010 | t            | -1879048192   | -1744830465
 test         |  102011 | t            | -1744830464   | -1610612737
 test         |  102012 | t            | -1610612736   | -1476395009
 test         |  102013 | t            | -1476395008   | -1342177281
 test         |  102014 | t            | -1342177280   | -1207959553
 test         |  102015 | t            | -1207959552   | -1073741825
 test         |  102016 | t            | -1073741824   | -939524097
 test         |  102017 | t            | -939524096    | -805306369
 test         |  102018 | t            | -805306368    | -671088641
 test         |  102019 | t            | -671088640    | -536870913
 test         |  102020 | t            | -536870912    | -402653185
 test         |  102021 | t            | -402653184    | -268435457
 test         |  102022 | t            | -268435456    | -134217729
 test         |  102023 | t            | -134217728    | -1
 test         |  102024 | t            | 0             | 134217727
 test         |  102025 | t            | 134217728     | 268435455
 test         |  102026 | t            | 268435456     | 402653183
 test         |  102027 | t            | 402653184     | 536870911
 test         |  102028 | t            | 536870912     | 671088639
 test         |  102029 | t            | 671088640     | 805306367
 test         |  102030 | t            | 805306368     | 939524095
 test         |  102031 | t            | 939524096     | 1073741823
 test         |  102032 | t            | 1073741824    | 1207959551
 test         |  102033 | t            | 1207959552    | 1342177279
 test         |  102034 | t            | 1342177280    | 1476395007
 test         |  102035 | t            | 1476395008    | 1610612735
 test         |  102036 | t            | 1610612736    | 1744830463
 test         |  102037 | t            | 1744830464    | 1879048191
 test         |  102038 | t            | 1879048192    | 2013265919
 test         |  102039 | t            | 2013265920    | 2147483647
(32 rows)

postgres=# select count(*) from test;
  count
---------
 1000000
(1 row)
```

#### Подключаюсь к другой ноде
```
kubectl get all
NAME                 READY   STATUS    RESTARTS   AGE
pod/citus-master-0   1/1     Running   0          26m
pod/citus-worker-0   1/1     Running   0          25m
pod/citus-worker-1   1/1     Running   0          24m
pod/citus-worker-2   1/1     Running   0          24m

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)          AGE
service/citus-master    LoadBalancer   10.96.181.166   158.160.130.204   5432:31708/TCP   26m
service/citus-workers   ClusterIP      None            <none>            5432/TCP         25m
service/kubernetes      ClusterIP      10.96.128.1     <none>            443/TCP          31m

NAME                            READY   AGE
statefulset.apps/citus-master   1/1     26m
statefulset.apps/citus-worker   3/3     25m


kubectl exec -it pod/citus-worker-2 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@citus-worker-2:/# psql -U postgres
psql (16.1 (Debian 16.1-1.pgdg120+1))
Type "help" for help.

postgres=# select count(*) from test;
  count
---------
 1000000
(1 row)
```

>если делать NodePort то kubectl port-forward service/postgres 5432:5432
