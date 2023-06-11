0. Развернул ВМ в Яндекс-облаке и подключился командой:
ssh -i ~/.ssh/yc_key otus@158.160.71.71
1. Установил постгрес 14 командой:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
Кластер стартовал:
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
2. Зашел в созданный кластер под пользователем postgres:
sudo -u postgres psql
3. Создал новую БД testdb
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
4. Зашел в базу testdb под пользователем postgres
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
5. Создал схему testnm.
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
6. Создал таблицу t1  - она создалась в схеме «public» (по умолчанию таблицы и другие объекты автоматически помещаются в схему «public». Она содержится во всех создаваемых базах данных)
testdb=# CREATE TABLE t1(c1 integer);
CREATE TABLE
7. Добавил в таблицу t1 запись:
testdb=# INSERT INTO t1 values(1);
INSERT 0 1
8. Создал новую роль с именем readonly:
testdb=# CREATE role readonly;
CREATE ROLE
9. Дал роли readonly право на подключение к базе данных testdb
testdb=# grant connect on DATABASE testdb TO readonly;
GRANT
10. Дал роли readonly право на использование схемы testnm
testdb=# grant usage on SCHEMA testnm to readonly;
GRANT
11. Дал роли readonly право на select для всех таблиц схемы testnm
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT
12. Создал пользователя testread с паролем test123
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
13. Дал роль readonly пользователю testread
testdb=# grant readonly TO testread;
GRANT ROLE
14. Зашел под пользователем testread в базу данных testdb (со второй попытки).
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
testdb=# psql -h 127.0.0.1 -U testread -d testdb -W
testdb-# \q
otus@otus-db-pg-vm-1:~$ psql -h 127.0.0.1 -U testread -d testdb -W
Password:
psql (14.8 (Ubuntu 14.8-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
testdb=>
15. Пробую сделать select * from t1;
testdb=> select * from t1;
ERROR:  permission denied for table t1
16. Не получилось - у пользователя отсутствуют права на выборку данных из таблицы t1.
17. Таблица t1 отсутствует в схеме testnm, находится в схеме public.
18. Права роли readonly были выданы на чтение данных в схеме testnm.
19. Получаю список таблиц:
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
20. Именно так! )
21. Спасибо! Учту возможность подобного нюанса...
22. Зашел в базу testdb под пользователем postgres
postgres=# \c testdb postgres
You are now connected to database "testdb" as user "postgres".
23. Удалил таблицу t1 (из схемы public)
testdb=# drop TABLE t1;
DROP TABLE
24. Создаю заново таблицу t1, но уже с явным указанием имени схемы testnm.
testdb=# CREATE TABLE testnm.t1(c1 integer);
CREATE TABLE
25. Добавил в таблицу t1 запись:
testdb=# INSERT INTO testnm.t1 values(1);
INSERT 0 1
26. Зашел под пользователем testread в базу данных testdb (аналогично п.14).
27. Пытаюсь сделать селект:
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
28. Не получилось.
29. Очень интересная особенность постгреса!
30. Применю гранты для вновь создаваемых таблиц
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly;
ALTER DEFAULT PRIVILEGES
31. Пытаюсь сделать селект:
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
32. Опять не получилось (
33. Пересоздал таблицу и теперь нужные права есть. Ура.
testdb=> select * from testnm.t1;
 c1
----
(0 rows)
34. Выполнил команду, таблица создалась и запись добавилась.
testdb=# create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
35. Как так?
36. Уяснил, что это все потому что search_path указывает в первую очередь на схему public. А схема public создается в каждой базе данных по умолчанию. И grant на все действия в этой схеме дается роли public. А роль public добавляется всем новым пользователям. Соответсвенно каждый пользователь может по умолчанию создавать объекты в схеме public любой базы данных, ес-но если у него есть право на подключение к этой базе данных. Чтобы раз и навсегда забыть про роль public - а в продакшн базе данных про нее лучше забыть - выполняем следующие действия: 
37. Выполняю команды из под postgres и заново коннечусь под testread (действовал по шпаргалке)
testdb=# revoke CREATE on SCHEMA public FROM public;
revoke all on DATABASE testdb FROM public;
REVOKE
REVOKE
38. Выполняю команды под пользователем testread:
testdb=> create table t3(c1 integer); insert into t3 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
ERROR:  relation "t3" does not exist
LINE 1: insert into t3 values (2);
39. Тут выполняется ограничение обычных пользователей личными схемами, отдельные права можно будет дополнительно назначать через принадлежность пользователей к ролям:
revoke CREATE on SCHEMA public FROM public;