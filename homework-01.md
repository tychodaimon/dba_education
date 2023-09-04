# Уровни изоляции транзакций

>Стандарт SQL допускает четыре уровня изоляции, которые определяются в терминах аномалий, которые допускаются при конкурентном выполнении транзакций на этом уровне:  
- «Грязное» чтение (dirty read). Транзакция T1 может читать строки измененные, но еще не зафиксированные, транзакцией T2. Отмена изменений (ROLLBACK) в T2 приведет к тому, что T1 прочитает данные, которых никогда не существовало.  
- Неповторяющееся чтение (non-repeatable read). После того, как транзакция T1 прочитала строку, транзакция T2 изменила или удалила эту строку и зафиксировала изменения (COMMIT). При повторном чтении этой же строки транзакция T1 видит, что строка изменена или удалена.  
- Фантомное чтение (phantom read). Транзакция T1 прочитала набор строк по некоторому условию. Затем транзакция T2 добавила строки, также удовлетворяющие этому условию. Если транзакция T1 повторит запрос, она получит другую выборку строк.  
- Аномалия сериализации - Результат успешной фиксации группы транзакций оказывается несогласованным при всевозможных вариантах исполнения этих транзакций по очереди.  

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

*Видим что пока таблица не закоммичена - она не видна во второй транзакции*
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
postgres=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres=*#
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
*Видим что строка sergey sergeev не появилась в второй транзакции, потому что на уровне изоляции read committed нет «Грязного» чтения?*
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
*Видим что строка sergey sergeev появилась в второй транзакции, потому что на уровне изоляции read committed есть неповторяющееся чтение?*


## Начинаем новые транзакции, в этот раз repeatable read 
### 1 терминал:
```
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
postgres=# set transaction isolation level repeatable read;
SET
postgres=*#
```
### 2 терминал:
```
postgres=# \set AUTOCOMMIT OFF
postgres=# \echo :AUTOCOMMIT
OFF
postgres=# set transaction isolation level repeatable read;
SET
postgres=*#
```

## В первой сессии добавляем новую запись, во второй делаем select
### 1 терминал:
```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
postgres=*#
```
### 2 терминал:
```
postgres=*#  select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*#
```
*Видим что строка sveta svetova не появилась в второй транзакции, потому что на уровне изоляции repeatable read нет «Грязного» чтения?*
### 1 терминал:
```
postgres=*# commit;
COMMIT
postgres=#
```
### 2 терминал:
```
postgres=*#  select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=*#
```
*Видим что строка sveta svetova не появилась в второй транзакции, потому что на уровне изоляции repeatable read в Postgres нет Фантомного чтения?*

## Завершаем вторую транзакцию и делаем повторный select
### 2 терминал:
```
postgres=*# commit;
COMMIT
postgres=#  select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)

postgres=*#
```
*Видим что строка sveta svetova появилась в второй транзакции, потому что обе транзакции применены*

Материалы:
- [Upgrading PostgreSQL Extensions](https://www.percona.com/blog/upgrading-postgresql-extensions/)  
- [PostgreSQL Database Security: External Server-Based Authentication](https://www.percona.com/blog/postgresql-database-security-external-server-based-authentication/)  
- [Lesser Known PostgreSQL Features](https://hakibenita.com/postgresql-unknown-features?ref=refind)  
- [Implementing event sourcing using a relational database](https://softwaremill.com/implementing-event-sourcing-using-a-relational-database/)  

- [Что такое Markdown и как им пользоваться](https://lifehacker.ru/chto-takoe-markdown/)  
- [https://habr.com/ru/articles/537594/](https://habr.com/ru/articles/537594/)  
