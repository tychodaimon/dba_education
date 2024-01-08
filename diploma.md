# Облачные кластеры и оптимизация производительности распределенного облака высокой доступности PostgreSQL

#### Устанавливаем вспомогательное программное обеспечение
```
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
ssh connect
yc init
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
sudo apt install postgresql-client
yc managed-kubernetes cluster get-credentials dba12 --external
kubectl cluster-info --kubeconfig /home/dba/.kube/config
kubectl config view
yc container cluster list
yc container cluster list-node-groups cat5fd4rtcnqqnohhu2e
yc compute disk list
kubectl get all --ignore-not-found
kubectl get nodes --ignore-not-found
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

#### Добавляем репу и ставим Zalando Postgres Operator и gui
```
helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator
helm install postgres-operator postgres-operator-charts/postgres-operator
helm repo add postgres-operator-ui-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator-ui
helm install postgres-operator-ui postgres-operator-ui-charts/postgres-operator-ui
```

#### Проверяем что все завелось
```
kubectl api-resources | grep postgres
kubectl get all
kubectl get deployments
kubectl describe deployments
```

#### Обеспечиваем доступность gui извне
```
#kubectl port-forward svc/postgres-operator-ui 8081:80
kubectl expose deployment postgres-operator-ui --type=LoadBalancer --name=my-service
kubectl get services my-service
```
>NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
>service/my-service             LoadBalancer   10.96.251.132   158.160.132.42   8081:30569/TCP

#### Через web-интерфейс выбираем параметры кластера
>Cluster YAML:
```
kind: "postgresql"
apiVersion: "acid.zalan.do/v1"

metadata:
  name: "minimal"
  namespace: "default"
  labels:
    team: acid

spec:
  teamId: "acid"
  postgresql:
    version: "15"
  numberOfInstances: 2
  enableMasterPoolerLoadBalancer: true
  enableReplicaPoolerLoadBalancer: true
  volume:
    size: "10Gi"
  users:
    test: []
  databases:
    otus: test
  allowedSourceRanges:
    # IP ranges to access your cluster go here
  
  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 500m
      memory: 500Mi
```

#### Проверяем конфигурацию
```
kubectl get all -A
NAMESPACE     NAME                                        READY   STATUS    RESTARTS       AGE
default       pod/minimal-0                               1/1     Running   0              8m58s
default       pod/minimal-1                               1/1     Running   0              8m11s
default       pod/postgres-operator-76b79dd5f8-cpdb9      1/1     Running   2 (20h ago)    22h
default       pod/postgres-operator-ui-7c4d9f9c45-wm7dw   1/1     Running   2 (20h ago)    22h
kube-system   pod/coredns-7c4497977d-7gwzp                1/1     Running   3 (20h ago)    46h
kube-system   pod/coredns-7c4497977d-mtnlj                1/1     Running   3 (20h ago)    46h
kube-system   pod/ip-masq-agent-q5hn2                     1/1     Running   3 (20h ago)    46h
kube-system   pod/ip-masq-agent-sgphm                     1/1     Running   3 (20h ago)    46h
kube-system   pod/ip-masq-agent-t2m9g                     1/1     Running   3 (20h ago)    46h
kube-system   pod/kube-dns-autoscaler-689576d9f4-zklc2    1/1     Running   3 (20h ago)    46h
kube-system   pod/kube-proxy-dss42                        1/1     Running   3 (20h ago)    46h
kube-system   pod/kube-proxy-nbgj2                        1/1     Running   3 (20h ago)    46h
kube-system   pod/kube-proxy-qr9sm                        1/1     Running   3 (20h ago)    46h
kube-system   pod/metrics-server-75d8b888d8-59hsm         2/2     Running   6 (20h ago)    46h
kube-system   pod/npd-v0.8.0-ft2ng                        1/1     Running   3 (20h ago)    46h
kube-system   pod/npd-v0.8.0-lqxdz                        1/1     Running   3 (20h ago)    46h
kube-system   pod/npd-v0.8.0-q25hc                        1/1     Running   3 (20h ago)    46h
kube-system   pod/yc-disk-csi-node-v2-5tnt9               6/6     Running   18 (20h ago)   46h
kube-system   pod/yc-disk-csi-node-v2-cq4nk               6/6     Running   18 (20h ago)   46h
kube-system   pod/yc-disk-csi-node-v2-s9xgf               6/6     Running   18 (20h ago)   46h

NAMESPACE     NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                  AGE
default       service/kubernetes             ClusterIP      10.96.128.1     <none>           443/TCP                  46h
default       service/minimal                ClusterIP      10.96.237.61    <none>           5432/TCP                 8m58s
default       service/minimal-config         ClusterIP      None            <none>           <none>                   8m9s
default       service/minimal-repl           ClusterIP      10.96.207.254   <none>           5432/TCP                 8m58s
default       service/my-service             LoadBalancer   10.96.251.132   158.160.132.42   8081:30569/TCP           21h
default       service/postgres-operator      ClusterIP      10.96.245.142   <none>           8080/TCP                 22h
default       service/postgres-operator-ui   ClusterIP      10.96.168.226   <none>           80/TCP                   22h
kube-system   service/kube-dns               ClusterIP      10.96.128.2     <none>           53/UDP,53/TCP,9153/TCP   46h
kube-system   service/metrics-server         ClusterIP      10.96.221.145   <none>           443/TCP                  46h

NAMESPACE     NAME                                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                                                        AGE
kube-system   daemonset.apps/ip-masq-agent                    3         3         3       3            3           beta.kubernetes.io/os=linux,node.kubernetes.io/masq-agent-ds-ready=true              46h
kube-system   daemonset.apps/kube-proxy                       3         3         3       3            3           kubernetes.io/os=linux,node.kubernetes.io/kube-proxy-ds-ready=true                   46h
kube-system   daemonset.apps/npd-v0.8.0                       3         3         3       3            3           beta.kubernetes.io/os=linux,node.kubernetes.io/node-problem-detector-ds-ready=true   46h
kube-system   daemonset.apps/nvidia-device-plugin-daemonset   0         0         0       0            0           beta.kubernetes.io/os=linux,node.kubernetes.io/nvidia-device-plugin-ds-ready=true    46h
kube-system   daemonset.apps/yc-disk-csi-node                 0         0         0       0            0           <none>                                                                               46h
kube-system   daemonset.apps/yc-disk-csi-node-v2              3         3         3       3            3           yandex.cloud/pci-topology=k8s                                                        46h

NAMESPACE     NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
default       deployment.apps/postgres-operator      1/1     1            1           22h
default       deployment.apps/postgres-operator-ui   1/1     1            1           22h
kube-system   deployment.apps/coredns                2/2     2            2           46h
kube-system   deployment.apps/kube-dns-autoscaler    1/1     1            1           46h
kube-system   deployment.apps/metrics-server         1/1     1            1           46h

NAMESPACE     NAME                                              DESIRED   CURRENT   READY   AGE
default       replicaset.apps/postgres-operator-76b79dd5f8      1         1         1       22h
default       replicaset.apps/postgres-operator-ui-7c4d9f9c45   1         1         1       22h
kube-system   replicaset.apps/coredns-7c4497977d                2         2         2       46h
kube-system   replicaset.apps/kube-dns-autoscaler-689576d9f4    1         1         1       46h
kube-system   replicaset.apps/metrics-server-64d75c78c6         0         0         0       46h
kube-system   replicaset.apps/metrics-server-75d8b888d8         1         1         1       46h

NAMESPACE   NAME                       READY   AGE
default     statefulset.apps/minimal   2/2     8m58s

NAMESPACE   NAME                                                    IMAGE                             CLUSTER-LABEL   SERVICE-ACCOUNT   MIN-INSTANCES   AGE
default     operatorconfiguration.acid.zalan.do/postgres-operator   ghcr.io/zalando/spilo-15:3.0-p1   cluster-name    postgres-pod      -1              22h

NAMESPACE   NAME                               TEAM   VERSION   PODS   VOLUME   CPU-REQUEST   MEMORY-REQUEST   AGE     STATUS
default     postgresql.acid.zalan.do/minimal   acid   15        2      10Gi     100m          100Mi            8m59s   Running

# kubectl get all -o wide

kubectl get postgresql
NAME      TEAM   VERSION   PODS   VOLUME   CPU-REQUEST   MEMORY-REQUEST   AGE   STATUS
minimal   acid   15        2      10Gi     100m          100Mi            12m   Running

kubectl get pods -l application=spilo -L spilo-role
NAME        READY   STATUS    RESTARTS   AGE   SPILO-ROLE
minimal-0   1/1     Running   0          21m   master
minimal-1   1/1     Running   0          20m   replica
```

#### Подключаемся к Postgres
```
kubectl get secret
NAME                                                    TYPE                 DATA   AGE
postgres.minimal.credentials.postgresql.acid.zalan.do   Opaque               2      23m
sh.helm.release.v1.postgres-operator-ui.v1              helm.sh/release.v1   1      22h
sh.helm.release.v1.postgres-operator.v1                 helm.sh/release.v1   1      22h
standby.minimal.credentials.postgresql.acid.zalan.do    Opaque               2      23m
test.minimal.credentials.postgresql.acid.zalan.do       Opaque               2      23m

export PGPASSWORD=$(kubectl get secret postgres.minimal.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d)
echo $PGPASSWORD
2G2bKz29KARBa2oeH4Ly0HzpIxnqmsxz06EHgjE5d4Bk0yizWWkGnW2jHuLazISd

export PGSSLMODE=require
export PGPORT="5432"
export PGHOST="10.96.237.61" # service/minimal ClusterIP

dba@cl1lagqbevptmgf1aij4-otat:~$ psql -U postgres
psql (12.16 (Ubuntu 12.16-0ubuntu0.20.04.1), server 15.2 (Ubuntu 15.2-1.pgdg22.04+1))
WARNING: psql major version 12, server major version 15.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | test     | UTF8     | en_US.utf-8 | en_US.utf-8 |
 postgres  | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
 template0 | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
postgres=# \q

```

#### Заходим в Pod
```

kubectl exec -it pod/minimal-0 -- bash

 ____        _ _
/ ___| _ __ (_) | ___
\___ \| '_ \| | |/ _ \
 ___) | |_) | | | (_) |
|____/| .__/|_|_|\___/
      |_|

This container is managed by runit, when stopping/starting services use sv

Examples:

sv stop cron
sv restart patroni

Current status: (sv status /etc/service/*)

run: /etc/service/patroni: (pid 33) 2124s
run: /etc/service/pgqd: (pid 32) 2124s

root@minimal-0:/home/postgres# psql -U postgres
psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
Type "help" for help.

postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 otus      | test     | UTF8     | en_US.utf-8 | en_US.utf-8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)
```

#### Добавил Pooler в Cluster YAML:
```
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  labels:
    team: acid
  name: minimal
  namespace: default
spec:
  allowedSourceRanges: []
  databases:
    otus: test
  enableConnectionPooler: true
  enableMasterPoolerLoadBalancer: true
  enableReplicaConnectionPooler: true
  enableReplicaPoolerLoadBalancer: true
  numberOfInstances: 2
  postgresql:
    version: '15'
  resources:
    limits:
      cpu: 500m
      memory: 500Mi
    requests:
      cpu: 100m
      memory: 100Mi
  teamId: acid
  users:
    test: []
  volume:
    iops: 3000
    size: 10Gi
    throughput: 125
```

#### Проверяем конфигурацию
```
kubectl get pods
NAME                                    READY   STATUS    RESTARTS      AGE
minimal-0                               1/1     Running   0             53m
minimal-1                               1/1     Running   0             52m
minimal-pooler-b54f88879-25tgn          1/1     Running   0             26s
minimal-pooler-b54f88879-5mstn          1/1     Running   0             26s
minimal-pooler-repl-7b5ccd94b5-5pcwx    1/1     Running   0             25s
minimal-pooler-repl-7b5ccd94b5-sd5jq    1/1     Running   0             25s
postgres-operator-76b79dd5f8-cpdb9      1/1     Running   2 (21h ago)   23h
postgres-operator-ui-7c4d9f9c45-wm7dw   1/1     Running   2 (21h ago)   23h


export PGPASSWORD=$(kubectl get secret postgres.minimal.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d)
echo $PGPASSWORD
2G2bKz29KARBa2oeH4Ly0HzpIxnqmsxz06EHgjE5d4Bk0yizWWkGnW2jHuLazISd

kubectl exec -it minimal-pooler-b54f88879-25tgn -- psql -U postgres -h localhost sslmode=require -W
Password:
psql (14.7, server 15.2 (Ubuntu 15.2-1.pgdg22.04+1))
WARNING: psql major version 14, server major version 15.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```

#### Убиваем мастера и смотрим что все переключилось
```
kubectl delete pod/minimal-0
pod "minimal-0" deleted


kubectl exec -it pod/minimal-1 -- bash

 ____        _ _
/ ___| _ __ (_) | ___
\___ \| '_ \| | |/ _ \
 ___) | |_) | | | (_) |
|____/| .__/|_|_|\___/
      |_|

This container is managed by runit, when stopping/starting services use sv

Examples:

sv stop cron
sv restart patroni

Current status: (sv status /etc/service/*)

run: /etc/service/patroni: (pid 24) 21s
run: /etc/service/pgqd: (pid 25) 21s
root@minimal-0:/home/postgres# patronictl -c postgres.yml list
+ Cluster: minimal ---------+---------+---------+----+-----------+
| Member    | Host          | Role    | State   | TL | Lag in MB |
+-----------+---------------+---------+---------+----+-----------+
| minimal-0 | 10.112.129.17 | Replica | running |  2 |         0 |
| minimal-1 | 10.112.128.14 | Leader  | running |  2 |           |
+-----------+---------------+---------+---------+----+-----------+
```

#### Тестируем и тюним настройки Patorni
```
kubectl exec -it pod/minimal-1 -- bash


pgbench -U postgres otus -i -s 100
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 58.34 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 204.47 s (drop tables 0.00 s, create tables 0.12 s, client-side generate 58.45 s, vacuum 62.64 s, primary keys 83.26 s).


pgbench -U postgres otus -t 5000 -c 4 -j 4
pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
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
latency average = 35.129 ms
initial connection time = 4.769 ms
tps = 113.866923 (without initial connection time)


patronictl -c /home/postgres/postgres.yml edit-config


failsafe_mode: false
loop_wait: 10
maximum_lag_on_failover: 33554432
postgresql:
  parameters:
    archive_mode: 'on'
    archive_timeout: 1800s
    autovacuum_analyze_scale_factor: 0.02
    autovacuum_max_workers: 5
    autovacuum_vacuum_scale_factor: 0.05
    bgwriter_delay: 200ms
    bgwriter_flush_after: 0
    bgwriter_lru_maxpages: 100
    bgwriter_lru_multiplier: 2.0
    checkpoint_completion_target: 0.9
    checkpoint_timeout: 15 min
    effective_cache_size: 11 GB
    effective_io_concurrency: 200
    enable_partitionwise_aggregate: true
    enable_partitionwise_join: true
    hot_standby: 'on'
    huge_pages: false
    jit: true
    log_autovacuum_min_duration: 0
    log_checkpoints: 'on'
    log_connections: 'on'
    log_disconnections: 'on'
    log_line_prefix: '%t [%p]: [%l-1] %c %x %d %u %a %h '
    log_lock_waits: 'on'
    log_min_duration_statement: 500
    log_statement: ddl
    log_temp_files: 0
    maintenance_io_concurrency: 200
    maintenance_work_mem: 320 MB
    max_connections: 100
    max_parallel_maintenance_workers: 2
    max_parallel_workers: 4
    max_parallel_workers_per_gather: 2
    max_replication_slots: 10
    max_slot_wal_keep_size: 1000 MB
    max_wal_senders: 10
    max_wal_size: 1024 MB
    max_worker_processes: 4
    min_wal_size: 512 MB
    parallel_leader_participation: true
    random_page_cost: 1.25
    shared_buffers: 4096 MB
    synchronous_commit: false
    tcp_keepalives_idle: 900
    tcp_keepalives_interval: 100
    track_functions: all
    track_wal_io_timing: true
    wal_buffers: -1
    wal_compression: 'on'
    wal_keep_size: 3650 MB
    wal_level: hot_standby
    wal_log_hints: 'on'
    wal_recycle: true
    work_mem: 32 MB
  use_pg_rewind: true
  use_slots: true
retry_timeout: 10
ttl: 30


patronictl -c /home/postgres/postgres.yml restart minimal


pgbench -U postgres otus -t 5000 -c 4 -j 4
pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
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
latency average = 25.923 ms
initial connection time = 4.788 ms
tps = 154.305624 (without initial connection time)
```
