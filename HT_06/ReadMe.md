1. Создал ВМ в ЯО.  
2. Подкючился к ВМ командой:  
ssh -i ~/.ssh/yc_key otus@158.160.10.250  
3. Установил postgres командой:  
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15  
4. Запустил postgres:  
otus@otus-db-pg-vm-1:~$ sudo -u postgres psql  
5. Проверил значение параметра deadlock_timeout:  
postgres=# show deadlock_timeout;  
 deadlock_timeout  
------------------  
 1s  
(1 row)  
6. Установил значения параметров:  
deadlock_timeout = 200 - в журнал сообщений сбрасывается информация о блокировках, удерживаемых более 200 миллисекунд  
system set log_lock_waits = on - будет ли создаваться сообщение журнала, когда сеанс ожидает блокировки дольше deadlock_timeout  
postgres=# alter system set deadlock_timeout = 200;  
alter system set log_lock_waits = on;  
ALTER SYSTEM  
ALTER SYSTEM  
7. После установки параметров перезапускаю кластер:  
otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main restart  
8. Проверяю, что параметры изменились:  

postgres=# show deadlock_timeout;  
 deadlock_timeout  
------------------  
 200ms  
(1 row)  
 
postgres=# show log_lock_waits;  
 log_lock_waits  
----------------  
 on  
  
9. Создаю БД locks:  
postgres=# create database locks;  
CREATE DATABASE  

10. Подключаюсь к базе locks, создаю таблицу persons и наполняю ее данными:  
locks=# create table persons(id serial, first_name text, second_name text);  
insert into persons(first_name, second_name) values('ivan', 'ivanov');  
insert into persons(first_name, second_name) values('petr', 'petrov');  
insert into persons(first_name, second_name) values('lena', 'lenina');  

CREATE TABLE  
INSERT 0 1  
INSERT 0 1  


11. Запустил еще 2 подключения к ВМ ЯО и подключаюсь к созданной базе locks.  

12. Последовательно для всех трех подключений к ВМ начинаю транзакцию и выполняю UPDATE для таблицы  

locks=# begin;  
update persons set first_name = 'ХХХХХХ' where id = 1;  

13. Выполняю запрос выборки данных из pg_locks  
locks=*# select *  
from pg_locks;  
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid  |       mode       | granted | fastpath |           waitstart  
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+------+------------------+---------+----------+-------------------------------  
 relation      |    16388 |    16397 |      |       |            |               |         |       |          | 6/2                | 6425 | RowExclusiveLock | t       | t        |  
 virtualxid    |          |          |      |       | 6/2        |               |         |       |          | 6/2                | 6425 | ExclusiveLock    | t       | t        |  
 relation      |    16388 |    16397 |      |       |            |               |         |       |          | 5/4                | 6415 | RowExclusiveLock | t       | t        |  
 virtualxid    |          |          |      |       | 5/4        |               |         |       |          | 5/4                | 6415 | ExclusiveLock    | t       | t        |  
 relation      |    16388 |    12073 |      |       |            |               |         |       |          | 4/38               | 6362 | AccessShareLock  | t       | t        |  
 relation      |    16388 |    16397 |      |       |            |               |         |       |          | 4/38               | 6362 | RowExclusiveLock | t       | t        |  
 virtualxid    |          |          |      |       | 4/38       |               |         |       |          | 4/38               | 6362 | ExclusiveLock    | t       | t        |  
 transactionid |          |          |      |       |            |           743 |         |       |          | 5/4                | 6415 | ExclusiveLock    | t       | f        |  
 tuple         |    16388 |    16397 |    0 |     1 |            |               |         |       |          | 5/4                | 6415 | ExclusiveLock    | t       | f        |  
 transactionid |          |          |      |       |            |           744 |         |       |          | 6/2                | 6425 | ExclusiveLock    | t       | f        |  
 transactionid |          |          |      |       |            |           742 |         |       |          | 5/4                | 6415 | ShareLock        | f       | f        | 2023-06-25 14:58:06.662926+00  
 transactionid |          |          |      |       |            |           742 |         |       |          | 4/38               | 6362 | ExclusiveLock    | t       | f        |  
 tuple         |    16388 |    16397 |    0 |     1 |            |               |         |       |          | 6/2                | 6425 | ExclusiveLock    | f       | f        | 2023-06-25 14:58:20.311599+00  

14. Анализ результатов запроса:  
Вторая транзакция ждет завершения первой, а третья после завершения первой получит блокировку на обновляемую строку и бужет ждать завершения второй.  

14. После последовательного коммита в трех подключениях - блокировки исчезли:  
locks=# select *  
from pg_locks;  
  locktype  | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid  |      mode       | granted | fastpath | waitstart  
------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+------+-----------------+---------+----------+-----------  
 relation   |    16388 |    12073 |      |       |            |               |         |       |          | 4/39               | 6362 | AccessShareLock | t       | t        |  
 virtualxid |          |          |      |       | 4/39       |               |         |       |          | 4/39               | 6362 | ExclusiveLock   | t       | t        |  

15. Воспроизведение ситуации, взаимоблокировки 3-х транзакций: последовательно апдейт таблицы persons  

15.1 Подключение 1:  
locks=# begin;  
update persons set first_name = 'XXXX' where id = 1;  
BEGIN  
UPDATE 1  
locks=*#  

15.2 Подключение 2:  
locks=# begin;  
update persons set first_name = 'XXXX' where id = 2;  
BEGIN  
UPDATE 1  
locks=*#  

15.3 Подключение 3:  
locks=# begin;  
update persons set first_name = 'XXXX' where id = 3;  
BEGIN  
UPDATE 1  
locks=*#  

15.4 Подключение 1:  
locks=*# begin;  
update persons set first_name = 'YYYY' where id = 2;  
WARNING:  there is already a transaction in progress  
BEGIN  

15.5 Подключение 2:  
locks=*# begin;  
update persons set first_name = 'YYYY' where id = 3;  
WARNING:  there is already a transaction in progress  

15.6 Подключение 3:  
locks=*# begin;  
update persons set first_name = 'YYYY' where id = 1;  
WARNING:  there is already a transaction in progress  

16. Мне трудно представить случаи взаимоблокировки 2-х транзакций, выполняющих UPDATE одной и той же таблицы (без where). Более опытные коллеги с бОльшим опытом администрирования БД возможно с таким сталкивались.  
