# Postgres & Docker 

#### Ставим докер
```
docker-owner@dba3-docker:~$ sudo su -
root@dba3-docker:~# apt-get update
root@dba3-docker:~# sudo apt-get install ca-certificates curl gnupg
root@dba3-docker:~# sudo install -m 0755 -d /etc/apt/keyrings
root@dba3-docker:~# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
root@dba3-docker:~# chmod a+r /etc/apt/keyrings/docker.gpg
root@dba3-docker:~# echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
root@dba3-docker:~# sudo apt-get update
root@dba3-docker:~# apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
root@dba3-docker:~# apt-get install docker-compose
```

#### Готовим точки монтирования
```
root@dba3-docker:~# mkdir /var/lib/postgres
root@dba3-docker:~# touch /root/server_bash_history
root@dba3-docker:~# touch /root/server_psql_history
```

#### Готовим конфиг для docker-compose для сервера.
```
root@dba3-docker:~# vi ./docker-compose.yaml
root@dba3-docker:~# cat ./docker-compose.yaml
version: "3.5"

services:
  postgres:
    container_name: postgres
    image: postgres:14.4
    shm_size: 1g # might be useful if you experience issues with database locally (e.g. "could not resize shared memory segment")
    ports:
      - "5432:5432"
    volumes:
      - /var/lib/postgresql/data:/var/lib/postgresql/data:delegated
      - /root/server_bash_history:/root/.bash_history
      - /root/server_psql_history:/root/.psql_history
    networks:
      - common
    environment:
      - POSTGRES_USER=docker-owner
      - POSTGRES_PASSWORD=test1234
      - POSTGRES_DB=otus
      - POSTGRES_INITDB_ARGS=--data-checksums
      - TZ=Europe/Moscow

networks:
  common:
    driver: bridge


root@dba3-docker:~# docker-compose up -d
```

#### Добавляем клиента.
```
root@dba3-docker:~# docker-compose down
root@dba3-docker:~# touch /root/client_bash_history
root@dba3-docker:~# touch /root/client_psql_history
root@dba3-docker:~# vi ./docker-compose.yaml
root@dba3-docker:~# cat ./docker-compose.yaml
version: "3.5"

services:
  postgres:
    container_name: postgres
    image: postgres:14.4
    shm_size: 1g # might be useful if you experience issues with database locally (e.g. "could not resize shared memory segment")
    ports:
      - "5432:5432"
    volumes:
      - /var/lib/postgresql/data:/var/lib/postgresql/data:delegated
      - /root/server_bash_history:/root/.bash_history
      - /root/server_psql_history:/root/.psql_history
    networks:
      - common
    environment:
      - POSTGRES_USER=docker-owner
      - POSTGRES_PASSWORD=test1234
      - POSTGRES_DB=otus
      - POSTGRES_INITDB_ARGS=--data-checksums
      - TZ=Europe/Moscow

  pg_client:
    container_name: pg_client
    image: postgres:14.4
    shm_size: 1g # might be useful if you experience issues with database locally (e.g. "could not resize shared memory segment")
    ports:
      - "5433:5433"
    volumes:
      - /root/client_bash_history:/root/.bash_history
      - /root/client_psql_history:/root/.psql_history
    networks:
      - common
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=postgres
      - TZ=Europe/Moscow

networks:
  common:
    driver: bridge


root@dba3-docker:~# docker-compose up -d
```

#### Подключаемся из контейнера с клиентом к контейнеру с сервером и создаем таблицу с парой строк
```
root@dba3-docker:~# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED        STATUS       PORTS                                                 NAMES
0e7260ca8353   postgres:14.4   "docker-entrypoint.s…"   8 hours ago    Up 8 hours   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   pg_client
77e7a988e1ae   postgres:14.4   "docker-entrypoint.s…"   23 hours ago   Up 8 hours   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp             postgres
root@dba3-docker:~# docker exec -it 0e7260ca8353 bash

root@0e7260ca8353:/# psql -h postgres -U docker-owner -W otus
Password:
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

tus=# create table t_sample (id serial primary key, value text);
CREATE TABLE
otus=# insert into t_sample (value) values ('first'), ('second');
INSERT 0 2
otus=# select * from t_sample;
 id | value
----+--------
  1 | first
  2 | second
(2 rows)

otus=#
```

#### Подключаемся к контейнеру с сервером снаружи
```
root@osnova:/var/www/osnova# sudo su -
root@osnova:~# su - postgres
postgres@osnova:~$ psql -h 51.250.3.21 -U docker-owner -W otus
Пароль:
psql (14.8 (Ubuntu 14.8-1.pgdg20.04+1), сервер 14.4 (Debian 14.4-1.pgdg110+1))
Введите "help", чтобы получить справку.

otus=# select * from t_sample;
 id | value
----+--------
  1 | first
  2 | second
(2 строки)

otus=#
```

#### Удаляем контейнер с сервером
```
root@dba3-docker:~# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED        STATUS       PORTS                                                 NAMES
0e7260ca8353   postgres:14.4   "docker-entrypoint.s…"   8 hours ago    Up 8 hours   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   pg_client
77e7a988e1ae   postgres:14.4   "docker-entrypoint.s…"   23 hours ago   Up 8 hours   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp             postgres
root@dba3-docker:~# docker stop 77e7a988e1ae
77e7a988e1ae
root@dba3-docker:~# docker rm 77e7a988e1ae
77e7a988e1ae
root@dba3-docker:~# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED       STATUS       PORTS                                                 NAMES
0e7260ca8353   postgres:14.4   "docker-entrypoint.s…"   8 hours ago   Up 8 hours   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   pg_client
```

#### Восстанавливаем контейнер из docker-compose
```
root@dba3-docker:~# docker-compose up -d
WARNING: Found orphan containers (root_client_1) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
pg_client is up-to-date
Creating postgres ... done
root@dba3-docker:~# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED         STATUS              PORTS                                                 NAMES
de91eb303984   postgres:14.4   "docker-entrypoint.s…"   6 seconds ago   Up 4 seconds        0.0.0.0:5432->5432/tcp, :::5432->5432/tcp             postgres
0e7260ca8353   postgres:14.4   "docker-entrypoint.s…"   8 hours ago     Up About a minute   5432/tcp, 0.0.0.0:5433->5433/tcp, :::5433->5433/tcp   pg_client
```

#### Подключаемся из контейнера с клиентом к контейнеру с сервером и проверяем данные
```
root@dba3-docker:~# docker exec -it 0e7260ca8353 bash
root@2917bd0f59b2:/# psql -h postgres -U docker-owner -W otus
Password:
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

otus=# \dt
            List of relations
 Schema |   Name   | Type  |    Owner
--------+----------+-------+--------------
 public | t_sample | table | docker-owner
(1 row)

otus=# select * from t_sample;
 id | value
----+--------
  1 | first
  2 | second
(2 rows)
```
