1. Развернул ВМ в Яндекс-облаке.
2. Подключился 1-ой сессией командой:
ssh -i ~/.ssh/yc_key otus@158.160.26.19
3. Подключился 2-ой сессией командой:
ssh -i ~/.ssh/yc_key otus@158.160.26.19
4. Установил постгрес командой:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
5. Проверил, в обеих сессиях что кластер стартовал командой:

otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
6. Создал новую БД и подключился к ней:
CREATE DATABASE ISO;
\с iso

6. В первой сессии отключил автокоммит:
iso=# \set AUTOCOMMIT OFF
iso=# \echo :AUTOCOMMIT
OFF

7. В первой сессии создал таблицу и добавил новые записи:
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;
Из-за явного коммита записи видны из обеих сессией

iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

8. Текущий уровень изоляции:

iso=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

9. В первой сессии добавили запись:

iso=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
iso=*# (* - признак незавершенной транзакции)

Она не видна из второй сессии:

iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

Завершаем транзакцию в первой сессии - commit;
Теперь видим добавленную запись из второй сессии:

iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
(3 rows)

Запись не была видна до коммита и стала видна потому что в транзакции, работающей на уровне "read committed", запрос SELECT (без предложения FOR UPDATE/SHARE) видит только те данные, которые были зафиксированы до начала запроса; 
он никогда не увидит незафиксированных данных или изменений, внесённых в процессе выполнения запроса параллельными транзакциями.

10. Установили уровень изоляции "repeatable read".

iso=# set transaction isolation level repeatable read
iso-# ;
SET
iso=*# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read
(1 row)

11. В первой сессии добавили запись:

iso=# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
iso=*#

Запись не видна - потому что в режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции, но не видны незафиксированные данные и изменения, произведённые другими транзакциями в процессе выполнения данной транзакции.

12. После коммита в первой сессии данные видны во второй сессии:

iso=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  5 | sergey     | sergeev
  6 | sveta      | svetova
(4 rows)