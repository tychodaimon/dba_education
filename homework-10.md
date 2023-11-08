# Разворачиваем Postgres и Clickhouse и загружаем большие данные

#### Ставим postgres 
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add
sudo apt -y update
sudo apt -y install postgresql-14 pgloader screen unzip
vi /etc/postgresql/14/main/postgresql.conf
sudo vi /etc/postgresql/14/main/postgresql.conf
```

#### Настройки postgres
```
max_connections = 100
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 2GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 10485kB
huge_pages = off
min_wal_size = 4GB
max_wal_size = 16GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2
listen_addresses = '*'
```

#### Создаем базу и таблицу для данных
```
sudo pg_ctlcluster 14 main restart

dba@dba10-pg:~/temp$ sudo su -
root@dba10-pg:~# su - postgres
postgres@dba10-pg:~$ psql
postgres=# create database otus;
CREATE DATABASE

postgres@dba10-pg:~$ psql otus
otus=# CREATE TABLE public.t_cars
(
    name text,
    price double precision,
    current_price double precision,
    lowest_price double precision,
    msrp_price double precision,
    image text,
    item_url text,
    category_name text,
    category_url text,
    available boolean,
    in_promotion boolean,
    lowest_product_price double precision,
    on_sale boolean,
    free_delivery boolean,
    product_id bigint,
    review_count smallint,
    review_stars double precision,
    short_desc text,
    brand_name text,
    item_url_key text,
    parsed_at timestamp with time zone
);
CREATE TABLE
```

#### Скачиваем архив https://www.kaggle.com/datasets/govzegunns/mimovrste-2021-2023-item-prices (95Gb)
```
postgres@dba10-pg:~$ screen -t loader
postgres@dba10-pg:~$ curl 'https://storage.googleapis.com/kaggle-data-sets/3773709/6527550/bundle/archive.zip?X-Goog-Algorithm=GOOG4-RSA-SHA256&X-Goog-Credential=gcp-kaggle-com%40kaggle-161607.iam.gserviceaccount.com%2F20231106%2Fauto%2Fstorage%2Fgoog4_request&X-Goog-Date=20231106T135803Z&X-Goog-Expires=259200&X-Goog-SignedHeaders=host&X-Goog-Signature=5b83b83ab1b8cabecd2b501f243e1d3e522d729d57bc41a30588516f178bfa39943a4ac11a09c552a521a947a3ddea8b9a0a155b35008fabfd4ce86c9d776d03008c77a974455b2ea4b19be6c3e98c0e961fe31f4b398ae5c664b715449d69abeef57eec44eb88e90a1943fad70d5d54f6cb5cb1268e43e730d51d76331a14619673a2515f0e8a72ab7f9fd9217083fc1de6ffa636eef52a48a7afb65fa84bd93dae4676611f917dd6c0f2c1e9b9380fbdc97be79f6f19541f73d19ce890d6df3f9e161dbe204f56f29562de6d6e271a43a6f3c1f9bb54f66070f6f44482def1b6156e887918335c1c09ca132762d03baae8e98bebd543b2bd6a045ef5968fd9' -o ./archive.zip

postgres@dba10-pg:~$ unzip ./archive.zip
postgres@dba10-pg:~$ rm archive.zip
```

#### Заливаем данные в Postgres
```
postgres@dba10-pg:~$ date; pgloader --type csv --with truncate --with "fields terminated by ';'" ~/mimodump-dataset.csv postgres:///otus?tablename=t_cars; date
Mon Nov  6 05:32:16 PM UTC 2023
2023-11-06T17:32:17.016001Z LOG pgloader version "3.6.7~devel"
2023-11-06T17:32:17.020001Z LOG Data errors in '/tmp/pgloader/'
2023-11-06T17:32:21.936229Z ERROR PostgreSQL ["\"public\".\"t_cars\""] Database error 22P02: invalid input syntax for type double precision: "price"
CONTEXT: COPY t_cars, line 1, column price: "price"
2023-11-07T01:52:52.981680Z LOG report summary reset
             table name     errors       rows      bytes      total time
-----------------------  ---------  ---------  ---------  --------------
                  fetch          0          0                     0.004s
-----------------------  ---------  ---------  ---------  --------------
      "public"."t_cars"          2  161709868    88.3 GB    8h20m35.838s
-----------------------  ---------  ---------  ---------  --------------
        Files Processed          0          1                     0.016s
COPY Threads Completion          0          2               8h20m35.838s
-----------------------  ---------  ---------  ---------  --------------
      Total import time          2  161709868    88.3 GB    8h20m35.854s
Tue Nov  7 01:52:53 AM UTC 2023
```

#### Выполняем еденичный запрос в Postgres
```
postgres@dba10-pg:~$ psql otus -c '\timing' -c 'select category_name, brand_name, count(*) from t_cars group by 1,2 order by 3 desc, 1 asc nulls last, 2 asc nulls last limit 10;'
Timing is on.
         category_name          | brand_name |  count
--------------------------------+------------+---------
 knjige vse                     |            | 1797425
 okvirji slike                  | tulup.si   | 1219650
 vrtno pohistvo                 | Greatstore | 1206060
 garniture vrtnega pohistva $91 | Greatstore |  971479
 vsa obutev                     | Adidas     |  945440
 slike plakati znaki            | tulup.si   |  765834
 majice otroske                 | Gap        |  752214
 vrtni stoli pocivalniki $91    | shumee     |  722699
 garniture vrtnega pohistva $91 | shumee     |  631574
 poezija                        |            |  599851
(10 rows)

Time: 440344.730 ms (07:20.345)
```

#### Выполняем 10 одновременных запросов в Postgres
```
postgres@dba10-pg:~$ cat ./benchmark.sh
#!/bin/bash

function_to_fork() {

read -r -d '' QUERY <<- EOM
	select  category_name,
		brand_name,
		count(*)
	from t_cars
	group by 1,2
	order by 3 desc, 1 asc nulls last, 2 asc nulls last
	limit 1;
EOM

psql otus -c '\timing' -c "$QUERY"

}

date;
for (( a = 1; a < 10; a++ ))
{
    function_to_fork &
}
date;
exit;

postgres@dba10-pg:~$ ./benchmark.sh
Wed Nov  8 12:00:19 AM UTC 2023
Wed Nov  8 12:00:19 AM UTC 2023
postgres@dba10-pg:~$ Timing is on.

 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 471516.389 ms (07:51.516)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 472072.485 ms (07:52.072)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473092.617 ms (07:53.093)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473112.046 ms (07:53.112)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473193.556 ms (07:53.194)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473276.482 ms (07:53.276)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473372.137 ms (07:53.372)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473400.319 ms (07:53.400)
 category_name | brand_name |  count
---------------+------------+---------
 knjige vse    |            | 1797425
(1 row)

Time: 473439.009 ms (07:53.439)
```

#### Устанавливаем clickhouse
```
sudo apt install dirmngr ca-certificates software-properties-common apt-transport-https curl
gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
gpg --export --armor 8919F6BD2B48D754 | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/clickhouse-key.gpg
echo "deb [arch=amd64] https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt update
sudo apt install clickhouse-server clickhouse-client
vi /etc/clickhouse-server/config.xml
sudo systemctl start clickhouse-server
sudo systemctl enable clickhouse-server
sudo systemctl status clickhouse-server
```

#### Создаем базу данных и таблицу
```
dba10-ch :) CREATE DATABASE IF NOT EXISTS otus;

CREATE DATABASE IF NOT EXISTS otus


root@dba10-ch:~# clickhouse-client --password 1321 --user default --database otus
ClickHouse client version 23.10.1.1976 (official build).
Connecting to database otus at localhost:9000 as user default.
Connected to ClickHouse server version 23.10.1 revision 54466.

Warnings:
 * Delay accounting is not enabled, OSIOWaitMicroseconds will not be gathered. Check /proc/sys/kernel/task_delayacct

dba10-ch :) CREATE TABLE otus.t_cars
(
 name String,
 price Float32,
 current_price Float32,
 lowest_price Float32,
 msrp_price Float32,
 image String,
 item_url String,
 category_name String,
 category_url String,
 available Bool,
 in_promotion Bool,
 lowest_product_price Float32,
 on_sale Bool,
 free_delivery Bool,
 product_id UInt32,
 review_count UInt16,
 review_stars Float32,
 short_desc String,
 brand_name String,
 item_url_key String,
 parsed_at DateTime
)
ENGINE = MergeTree()
PRIMARY KEY (product_id)

CREATE TABLE otus.t_cars
(
    `name` String,
    `price` Float32,
    `current_price` Float32,
    `lowest_price` Float32,
    `msrp_price` Float32,
    `image` String,
    `item_url` String,
    `category_name` String,
    `category_url` String,
    `available` Bool,
    `in_promotion` Bool,
    `lowest_product_price` Float32,
    `on_sale` Bool,
    `free_delivery` Bool,
    `product_id` UInt32,
    `review_count` UInt16,
    `review_stars` Float32,
    `short_desc` String,
    `brand_name` String,
    `item_url_key` String,
    `parsed_at` DateTime
)
ENGINE = MergeTree
PRIMARY KEY product_id

Query id: fc643fd6-898d-4516-9788-0aa75606ca3a

Ok.

0 rows in set. Elapsed: 0.013 sec.

dba10-ch :)
```

#### Заливаем данные в ClickHouse
```
root@dba10-ch:~# date; clickhouse-client --password 1321 --user default --database otus -q "INSERT INTO otus.t_cars FORMAT CSVWithNames" --format_csv_delimiter ";" --date_time_input_format=best_effort < mimodump-dataset.csv ;date
Wed Nov  8 01:50:11 AM UTC 2023
Wed Nov  8 01:57:30 AM UTC 2023
```

#### Выполняем еденичный запрос в ClickHouse
```
root@dba10-ch:~# clickhouse client -h 127.0.0.1 --port 9000 --database otus -u default --password 1321 --time --query "select category_name, brand_name, count(*) from t_cars group by 1,2 order by 3 desc, 1 asc nulls last, 2 asc nulls last limit 10;"
knjige vse					1797431
okvirji slike	tulup.si			1219650
vrtno pohistvo	Greatstore			1206060
garniture vrtnega pohistva $91	Greatstore	971479
vsa obutev	Adidas				945440
slike plakati znaki	tulup.si		765834
majice otroske	Gap				752214
vrtni stoli pocivalniki $91	shumee		722699
garniture vrtnega pohistva $91	shumee		631574
poezija						599851
4.995s
```

#### Выполняем 10 одновременных запросов в ClickHouse
```
root@dba10-ch:~# cat ./benchmark.sh
#!/bin/bash

function_to_fork() {

read -r -d '' QUERY <<- EOM
	select category_name, brand_name, count(*)
	from t_cars
	group by 1,2
	order by 3 desc, 1 asc nulls last, 2 asc nulls last
	limit 3;
EOM

    clickhouse client -h 127.0.0.1 --port 9000 --database otus -u default --password 1321 --time --query "$QUERY" >> /dev/null;

    exit;
}

date;
for (( a = 1; a < 10; a++ ))
{
    function_to_fork &
}
date;
exit;

root@dba10-ch:~# ./benchmark.sh
Wed Nov  8 08:23:29 PM UTC 2023
Wed Nov  8 08:23:29 PM UTC 2023
root@dba10-ch:~# 48.123
48.950
48.912
49.004
49.224
49.248
49.240
49.166
49.328
```

#### В итоге можно сказать что CH намного быстрее принимает большой обьем данных и испособен намного быстрее выполнять запросы на этих данных, но его производительность дегражирует при большом колличестве одновременных запросов, PG обрабатывает данные намного медленнее (и залив и селекты) но меньше деградирует при одновременных обращениях (в определенных пределах естественно)
