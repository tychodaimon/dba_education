# развернуть multi master кластер CockroachDB своими руками

#### Устанавливаем CockroachDB на первую ноду
```
wget -qO- https://binaries.cockroachdb.com/cockroach-v23.2.0-alpha.6.linux-amd64.tgz | tar  xvz && sudo cp -i cockroach-v23.2.0-alpha.6.linux-amd64/cockroach /usr/local/bin/ && sudo mkdir -p /opt/cockroach && sudo chown dba14:dba14 /opt/cockroach

cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key

cockroach cert create-node localhost dba14cockroach1 dba14cockroach2 dba14cockroach3 --certs-dir=certs --ca-key=my-safe-directory/ca.key --overwrite

cockroach cert create-client root --certs-dir=certs --ca-key=my-safe-directory/ca.key

cockroach cert list --certs-dir=certs
Certificate directory: certs
  Usage  | Certificate File |    Key File     |  Expires   |                                Notes                                 | Error
---------+------------------+-----------------+------------+----------------------------------------------------------------------+--------
  CA     | ca.crt           |                 | 2033/11/25 | num certs: 1                                                         |
  Node   | node.crt         | node.key        | 2028/11/21 | addresses: localhost,dba14cockroach1,dba14cockroach2,dba14cockroach3 |
  Client | client.root.crt  | client.root.key | 2028/11/21 | user: root                                                           |
(3 rows)

sudo apt install zip unzip

zip -r certs.zip ./certs
  adding: certs/ (stored 0%)
  adding: certs/client.root.key (deflated 23%)
  adding: certs/node.key (deflated 23%)
  adding: certs/ca.crt (deflated 25%)
  adding: certs/client.root.crt (deflated 24%)
  adding: certs/node.crt (deflated 25%)

logout
```

#### Переносим сертификаты на другие ноды
```
user@daemaken ~ % scp dba14@158.160.35.78:~/certs.zip ./
certs.zip			100% 6224   742.8KB/s   00:00

user@daemaken ~ % scp ./certs.zip dba14@158.160.56.35:~/
certs.zip			100% 6224   702.7KB/s   00:00

user@daemaken ~ % scp ./certs.zip dba14@158.160.104.120:~/
certs.zip			100% 6224   639.0KB/s   00:00
```

#### Запускаем и инициализируем CockroachDB на первой ноде
```
sudo cockroach start --certs-dir=certs --advertise-addr=dba14cockroach1 --join=dba14cockroach1 --cache=.25 --max-sql-memory=.25 --background
*
* WARNING: Running a server without --sql-addr, with a combined RPC/SQL listener, is deprecated.
* This feature will be removed in a later version of CockroachDB.
*
*
* INFO: initial startup completed.
* Node will now attempt to join a running cluster, or wait for `cockroach init`.
* Client connections will be accepted after this completes successfully.
* Check the log file(s) for progress.
*

cockroach init --certs-dir=certs --host=dba14cockroach1
Cluster successfully initialized

cockroach node status --certs-dir=certs
  id |        address        |      sql_address      |      build      |              started_at              |              updated_at              | locality | is_available | is_live
-----+-----------------------+-----------------------+-----------------+--------------------------------------+--------------------------------------+----------+--------------+----------
   1 | dba14cockroach1:26257 | dba14cockroach1:26257 | v23.2.0-alpha.6 | 2023-11-18 21:14:32.738249 +0000 UTC | 2023-11-18 21:15:59.777684 +0000 UTC |          | true         | true
(1 row)
```

#### Создаем таблицу и заливаем данные
```
cockroach sql --certs-dir=certs
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v23.2.0-alpha.6 (x86_64-pc-linux-gnu, built 2023/11/07 17:06:25, go1.20.10 X:nocoverageredesign) (same version as client)
# Cluster ID: 643cc51e-51f3-4af5-ad96-9ca739097b0a
#
# Enter \? for a brief introduction.
#
root@localhost:26257/system/defaultdb>
root@localhost:26257/system/defaultdb> CREATE TABLE if not exists t_sales (
                                    ->     region text,
                                    ->     country text,
                                    ->     item_type text,
                                    ->     sales_channel text,
                                    ->     order_priority text,
                                    ->     order_date date not null,
                                    ->     order_id int not null,
                                    ->     ship_date date,
                                    ->     units_sold integer,
                                    ->     unit_price decimal(19,2),
                                    ->     unit_cost decimal(19,2),
                                    ->     total_revenue decimal(19,2),
                                    ->     total_cost decimal(19,2),
                                    ->     total_profit decimal(19,2)
                                    -> );
CREATE TABLE

Time: 24ms total (execution 24ms / network 0ms)

root@localhost:26257/system/defaultdb> ALTER TABLE t_sales ADD PRIMARY KEY (order_id, order_date);
ALTER TABLE

Time: 941ms total (execution 941ms / network 0ms)

root@localhost:26257/system/defaultdb> IMPORT INTO t_sales (Region,Country,Item_Type,Sales_Channel,Order_Priority,Order_Date,Order_ID,Ship_Date,Units_Sold,Unit_Price,Unit_Cost,Total_Revenue,Total_Cost,Total_Profit) \
CSV DATA ('https://storage.googleapis.com/postgres13/1000000SalesRecords.csv?AUTH=implicit') WITH DELIMITER = ',', SKIP = '1';
```

#### Подготавливаем остальные ноды кластера
```
wget -qO- https://binaries.cockroachdb.com/cockroach-v23.2.0-alpha.6.linux-amd64.tgz | tar  xvz && sudo cp -i cockroach-v23.2.0-alpha.6.linux-amd64/cockroach /usr/local/bin/ && sudo mkdir -p /opt/cockroach && sudo chown dba14:dba14 /opt/cockroach

sudo apt install zip unzip

unzip ./certs.zip
Archive:  ./certs.zip
   creating: certs/
  inflating: certs/client.root.key
  inflating: certs/node.key
  inflating: certs/ca.crt
  inflating: certs/client.root.crt
  inflating: certs/node.crt
```

#### Запускаем 2 и 3 ноды
```
sudo cockroach start --certs-dir=certs --advertise-addr=dba14cockroach2 --join=dba14cockroach1 --cache=.25 --max-sql-memory=.25 --background
sudo cockroach start --certs-dir=certs --advertise-addr=dba14cockroach3 --join=dba14cockroach1 --cache=.25 --max-sql-memory=.25 --background
```

#### Проверяем что все застегнулось на первой ноде
```
cockroach node status --certs-dir=certs
  id |        address        |      sql_address      |      build      |              started_at              |              updated_at              | locality | is_available | is_live
-----+-----------------------+-----------------------+-----------------+--------------------------------------+--------------------------------------+----------+--------------+----------
   1 | dba14cockroach1:26257 | dba14cockroach1:26257 | v23.2.0-alpha.6 | 2023-11-18 21:14:32.738249 +0000 UTC | 2023-11-18 23:37:44.771741 +0000 UTC |          | true         | true
   2 | dba14cockroach2:26257 | dba14cockroach2:26257 | v23.2.0-alpha.6 | 2023-11-18 23:35:41.679909 +0000 UTC | 2023-11-18 23:37:44.707328 +0000 UTC |          | true         | true
   3 | dba14cockroach3:26257 | dba14cockroach3:26257 | v23.2.0-alpha.6 | 2023-11-18 23:37:43.507644 +0000 UTC | 2023-11-18 23:37:46.521875 +0000 UTC |          | true         | true
(3 rows)
```

#### Проверяем результат на другой ноде (на 3-ей)
```
ba14@dba14cockroach3:~$ cockroach sql --certs-dir=certs
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v23.2.0-alpha.6 (x86_64-pc-linux-gnu, built 2023/11/07 17:06:25, go1.20.10 X:nocoverageredesign) (same version as client)
# Cluster ID: 643cc51e-51f3-4af5-ad96-9ca739097b0a
#
# Enter \? for a brief introduction.
#
root@localhost:26257/system/defaultdb> \dt
List of relations:
  Schema |  Name   | Type  | Owner
---------+---------+-------+--------
  public | t_sales | table | root
(1 row)
root@localhost:26257/system/defaultdb> select count(*) from t_sales;
  count
----------
  949991
(1 row)

Time: 444ms total (execution 443ms / network 0ms)

root@localhost:26257/system/defaultdb> select * from t_sales limit 1;
             region            |   country   | item_type | sales_channel | order_priority | order_date | order_id  | ship_date  | units_sold | unit_price | unit_cost | total_revenue | total_cost | total_profit
-------------------------------+-------------+-----------+---------------+----------------+------------+-----------+------------+------------+------------+-----------+---------------+------------+---------------
  Middle East and North Africa | Afghanistan | Baby Food | Offline       | C              | 2014-07-04 | 100001180 | 2014-07-30 |       1426 |     255.28 |    159.42 |     364029.28 |  227332.92 |    136696.36
(1 row)

Time: 1ms total (execution 1ms / network 0ms)

root@localhost:26257/system/defaultdb> select sum(total_profit) / count(*) as avg_profit from t_sales;
       avg_profit
-------------------------
  392282.69951175326924
(1 row)

Time: 792ms total (execution 791ms / network 0ms)
```
