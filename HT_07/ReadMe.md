1. Развернул ВМ в Яндекс-облаке и подключился командой:
ssh -i ~/.ssh/yc_key otus@158.160.15.159

2. Установил постгрес 15 командой:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
Кластер стартовал:
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

3. Зашел в созданный кластер под пользователем postgres:
otus@otus-db-pg-vm-1:~$ sudo -u postgres psql

4. Изменил параметры:
postgres=# alter system set checkpoint_timeout = 30;
ALTER SYSTEM
postgres=# alter system set log_checkpoints = on;
ALTER SYSTEM

5. Перечитал файл конфигурации:
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

6. Проверяю настройки:
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)

postgres=# show log_checkpoints;
 log_checkpoints
-----------------
 on
(1 row)

7. Инициализирую pgbench под пользователем postgres;
otus@otus-db-pg-vm-1:~$ sudo su - postgres
postgres@otus-db-pg-vm-1:~$ pgbench -i postgres;
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.80 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.22 s, vacuum 0.05 s, primary keys 0.51 s).

8. Текущий LSN (последовательный номер в журнале):
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/21C6488
(1 row)


9. Количество контрольных точек:
postgres=# select checkpoints_timed from pg_stat_bgwriter;
 checkpoints_timed
-------------------
                28
(1 row)

10. 10 минут c помощью утилиты pgbench подавал нагрузку:
postgres@otus-db-pg-vm-1:~$ pgbench -c 8 -P 60 -T 600 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 397.7 tps, lat 20.101 ms stddev 16.590, 0 failed
progress: 120.0 s, 375.4 tps, lat 21.316 ms stddev 16.741, 0 failed
progress: 180.0 s, 371.5 tps, lat 21.536 ms stddev 18.681, 0 failed
progress: 240.0 s, 386.3 tps, lat 20.705 ms stddev 18.725, 0 failed
progress: 300.0 s, 351.5 tps, lat 22.763 ms stddev 20.948, 0 failed
progress: 360.0 s, 409.0 tps, lat 19.562 ms stddev 19.128, 0 failed
progress: 420.0 s, 394.7 tps, lat 20.263 ms stddev 18.072, 0 failed
progress: 480.0 s, 370.9 tps, lat 21.568 ms stddev 18.532, 0 failed
progress: 540.0 s, 383.5 tps, lat 20.868 ms stddev 19.920, 0 failed
progress: 600.0 s, 370.8 tps, lat 21.571 ms stddev 19.973, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 228676
number of failed transactions: 0 (0.000%)
latency average = 20.990 ms
latency stddev = 18.771 ms
initial connection time = 16.520 ms
tps = 381.121931 (without initial connection time)

11. Считываю LSN и кол-во контрольных точек:

postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/1A9D1348
(1 row)

postgres=# select checkpoints_timed from pg_stat_bgwriter;
 checkpoints_timed
-------------------
                59
(1 row)

12. Количество байт между значениями LSN до и после запуска pgbench:

postgres=# select '0/1A9D1348'::pg_lsn - '0/21C6488'::pg_lsn as byte_size;
 byte_size
-----------
 411086528
(1 row)

13. Среднее значение, приходящееся на одну контрольную точку:

postgres=# select round(('0/1A9D1348'::pg_lsn - '0/21C6488'::pg_lsn) / (59-28)/1024/1024, 0) as Кbyte_size;
 Кbyte_size
------------
         13
(1 row)

14. Контрольные точки в журнале (вывожу первые 10 записей):

postgres@otus-db-pg-vm-1:~$ tail -n 10 /var/log/postgresql/postgresql-15-main.log | grep checkpoint
2023-06-27 16:13:38.077 UTC [5139] LOG:  checkpoint starting: time
2023-06-27 16:14:05.089 UTC [5139] LOG:  checkpoint complete: wrote 1827 buffers (11.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.901 s, sync=0.021 s, total=27.013 s; sync files=9, longest=0.012 s, average=0.003 s; distance=19054 kB, estimate=20186 kB
2023-06-27 16:14:08.091 UTC [5139] LOG:  checkpoint starting: time
2023-06-27 16:14:35.081 UTC [5139] LOG:  checkpoint complete: wrote 2055 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.892 s, sync=0.025 s, total=26.990 s; sync files=17, longest=0.014 s, average=0.002 s; distance=19387 kB, estimate=20106 kB
2023-06-27 16:14:38.084 UTC [5139] LOG:  checkpoint starting: time
2023-06-27 16:15:05.147 UTC [5139] LOG:  checkpoint complete: wrote 1860 buffers (11.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.971 s, sync=0.020 s, total=27.064 s; sync files=9, longest=0.016 s, average=0.003 s; distance=19235 kB, estimate=20019 kB
2023-06-27 16:15:08.151 UTC [5139] LOG:  checkpoint starting: time
2023-06-27 16:15:35.121 UTC [5139] LOG:  checkpoint complete: wrote 1814 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.886 s, sync=0.020 s, total=26.971 s; sync files=13, longest=0.013 s, average=0.002 s; distance=18773 kB, estimate=19895 kB
2023-06-27 16:15:38.124 UTC [5139] LOG:  checkpoint starting: time
2023-06-27 16:16:05.033 UTC [5139] LOG:  checkpoint complete: wrote 1790 buffers (10.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.878 s, sync=0.012 s, total=26.910 s; sync files=8, longest=0.007 s, average=0.002 s; distance=19095 kB, estimate=19815 kB

15. Разница между стартом и фиксацией контрольной точки = 27 секунд. Равна произведению параметров checkpoint_completion_target (равен 0.9) и checkpoint_timeout (равен 30).
postgres=# show checkpoint_completion_target;
 checkpoint_completion_target
------------------------------
 0.9
(1 row)

postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)

16. Проверил что при предыдущем запуске pgbench был включен режим синхронного коммита транзакций:
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

17. Исправил параметр synchronous_commit postgresql.conf
otus@otus-db-pg-vm-1:~$ sudo nano /etc/postgresql/15/main/postgresql.conf

synchronous_commit = off                # synchronization level;

18. Перезапустил кластер:
otus@otus-db-pg-vm-1:~$ sudo systemctl restart postgresql

19. Проверил значение параметра:

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 row)

20. Заново 10 минут c помощью утилиты pgbench подавал нагрузку:

postgres@otus-db-pg-vm-1:~$ pgbench -c 8 -P 60 -T 600 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 3210.3 tps, lat 2.491 ms stddev 0.755, 0 failed
progress: 120.0 s, 3246.7 tps, lat 2.463 ms stddev 0.905, 0 failed
progress: 180.0 s, 3212.1 tps, lat 2.490 ms stddev 0.738, 0 failed
progress: 240.0 s, 3205.3 tps, lat 2.495 ms stddev 0.739, 0 failed
progress: 300.0 s, 3165.9 tps, lat 2.526 ms stddev 0.755, 0 failed
progress: 360.0 s, 3215.8 tps, lat 2.487 ms stddev 0.782, 0 failed
progress: 420.0 s, 3277.2 tps, lat 2.440 ms stddev 0.705, 0 failed
progress: 480.0 s, 2895.4 tps, lat 2.762 ms stddev 0.893, 0 failed
progress: 540.0 s, 3120.0 tps, lat 2.563 ms stddev 0.807, 0 failed
progress: 600.0 s, 3194.1 tps, lat 2.504 ms stddev 0.742, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1904571
number of failed transactions: 0 (0.000%)
latency average = 2.519 ms
latency stddev = 0.788 ms
initial connection time = 15.657 ms
tps = 3174.265685 (without initial connection time)

21. Мы получили гораздо лучшие значения tps, потому что сервер не ждет пока записи из WAL сохранятся на диске, прежде чем сообщить клиенту об успешном завершении операции.
Но при применении данного параметра (synchronous_commit = off) отсутствуют:
- гарантированная локальная фиксация	
- гарантированная фиксация на ведомом после сбоя PG	
- гарантированная фиксация на ведомом после сбоя ОС	
- согласованность запросов на ведомом

22. Остановил кластер:

otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main stop

23. Создал новый кластер с помощью утилиты pg_createcluster. Контрольная сумма страниц включена

otus@otus-db-pg-vm-1:~$ sudo pg_createcluster 15 main2 -- --data-checksums
...
Ver Cluster Port Status Owner    Data directory               Log file
15  main2   5433 down   postgres /var/lib/postgresql/15/main2 /var/log/postgresql/postgresql-15-main2.log

24. Стартую кластер и подключаюсь к постгрес:

otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main2 start
postgres@otus-db-pg-vm-1:~$ psql -p 5433

25. Значение параметра data_checksums:

postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)

26. Создаю БД test, подключаюсь к ней:  
postgres=# create database test;
CREATE DATABASE
postgres=# \c test postgres
You are now connected to database "test" as user "postgres". 

27. Подключаюсь к базе test, создаю таблицу persons и наполняю ее данными:  
test=# create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
insert into persons(first_name, second_name) values('lena', 'lenina');
CREATE TABLE
INSERT 0 1
INSERT 0 1
INSERT 0 1

28. Проверяю наполнение таблицы:
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | lena       | lenina
(3 rows)

29. Путь к данным таблицы:

test=# select pg_relation_filepath('persons');
 pg_relation_filepath
----------------------
 base/16388/16390
(1 row)

30. Останавливаю кластер:

otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main2 stop

31. Изменяю файл:
otus@otus-db-pg-vm-1:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/main2/base/16388/16390 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00801671 s, 1.0 kB/s

32. Стартую кластер:
otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main2 start

33. Подключаюсь и делаю селект. Выдает ошибку контрольной суммы файла таблицы:

test=# select * from persons;
WARNING:  page verification failed, calculated checksum 40544 but expected 55109
ERROR:  invalid page in block 0 of relation base/16388/16390

34. Проверил значение параметра ignore_checksum_failure.
Если параметр ignore_checksum_failure включён, система игнорирует проблему (но всё же предупреждает о ней) и продолжает обработку. Это поведение может привести к краху, распространению или сокрытию повреждения данных и другим серьёзными проблемам. Однако, включив его, можно обойти ошибку и получить неповреждённые данные, которые могут находиться в таблице, если цел заголовок блока. Если же повреждён заголовок, будет выдана ошибка, даже когда этот параметр включён.

test=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 off
(1 row)

35. Присвоил значение ignore_checksum_failure = on:   

test=# set ignore_checksum_failure = on;   
SET   
test=# show ignore_checksum_failure;   
 ignore_checksum_failure   
-------------------------   
 on   
(1 row)   

36. Сделал селект с включенным параметром - ошибка выдалась, но данны проситать смогли:

test=# select * from persons;
WARNING:  page verification failed, calculated checksum 40544 but expected 55109
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | lena       | lenina
(3 rows )