# Уровни изоляции транзакций

|                  | «грязное» чтение        | неповторяющееся чтение | фантомное чтение        | аномалия сериализации |
| :---             |           :----:        |        :----:          |     :----:              |     :----:            |
| Read Uncommitted | Допускается, но не в PG | да                     | да                      | да                    |
| Read Сommitted   |             -           | да                     | да                      | да                    |
| Repeatable Read  |             -           | -                      | Допускается, но не в PG | да                    |
| Serializable     |             -           | -                      | -                       | -                     |

## Эксперимент: открываем 2 теминала
### 1 терминал:
```
id_ed25519@dba1:~$ sudo su -
root@dba1:~# su - postgres
postgres@dba1:~$ psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# \echo :AUTOCOMMIT  
on  
postgres=# \set AUTOCOMMIT OFF  
postgres=# \echo :AUTOCOMMIT  
OFF  
postgres=# create table persons(id serial, first_name text, second_name text);  
CREATE TABLE  
postgres=*# insert into persons(first_name, second_name) values('ivan', 'ivanov')  
postgres-*# ;  
INSERT 0 1  
postgres=*# insert into persons(first_name, second_name) values('petr', 'petrov');  
INSERT 0 1  
postgres=*# show transaction isolation level;  
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres=*#
```
### 2 терминал:
```
id_ed25519@dba1:~$ sudo su -
root@dba1:~# su - postgres
postgres@dba1:~$ psql
psql (15.4 (Ubuntu 15.4-1.pgdg22.04+1))
Type "help" for help.

postgres=# \echo :AUTOCOMMIT
on
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
postgres=# select * from persons;
ERROR:  relation "persons" does not exist
LINE 1: select * from persons;
                      ^
postgres=!#
```

Видим что пока таблица не закоммичена - она не видна во второй транзакции
## Коммитим транзакции:

### 1 терминал:
```
postgres=*# commit;
COMMIT
postgres=#
```
### 2 терминал:
```
postgres=!# select * from persons;
ERROR:  current transaction is aborted, commands ignored until end of transaction block
postgres=!#
```

## Проверяем текущий уровень изоляции:
### 1 терминал:
```
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres=*#
```
### 2 терминал:
```
postgres=!# show transaction isolation level;
ERROR:  current transaction is aborted, commands ignored until end of transaction block
postgres=!# rollback;
ROLLBACK
postgres=#
```
## добавляем новую запись
### 1 терминал:
```
postgres=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
postgres=*#
```
### 2 терминал:
```
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

postgres=*#
```
Видим что строка sergey sergeev не появилась в второй транзакции, потому что на уровне изоляции read committed нет «Грязного» чтения?
### 1 терминал:
```
postgres=*# commit;
COMMIT
postgres=#
```
### 2 терминал:
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*#
```
Видим что строка sergey sergeev появилась в второй транзакции, потому что на уровне изоляции read committed есть неповторяющееся чтение?

