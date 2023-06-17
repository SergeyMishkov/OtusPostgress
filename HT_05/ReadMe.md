1. Создал ВМ, установил постгрес :
2. Запустил утилиту pgbench. Результаты запуска:
postgres@otus-db-pg-vm-10:~$ pgbench -i postgres
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
done in 2.10 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.69 s, vacuum 0.04 s, primary keys 1.36 s).
Данной командой мы инициализировали базу данных в постгрес - создались таблицы:
pgbench_accounts
pgbench_branches
pgbench_history
pgbench_tellers
3. Запустили pgbench с параметрами:
-c8   - количество клиентов
-P 6  - показывать прогресс каждые 6 секунд
-T 60 - количество транзакций, выполняемых каждым клиентом

postgres@otus-db-pg-vm-10:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 587.2 tps, lat 13.570 ms stddev 8.668, 0 failed
progress: 12.0 s, 667.0 tps, lat 11.974 ms stddev 7.766, 0 failed
progress: 18.0 s, 404.3 tps, lat 19.813 ms stddev 23.885, 0 failed
progress: 24.0 s, 630.3 tps, lat 12.686 ms stddev 7.643, 0 failed
progress: 30.0 s, 448.7 tps, lat 17.827 ms stddev 57.843, 0 failed
progress: 36.0 s, 476.0 tps, lat 16.791 ms stddev 14.274, 0 failed
progress: 42.0 s, 483.2 tps, lat 16.581 ms stddev 11.467, 0 failed
progress: 48.0 s, 418.5 tps, lat 19.055 ms stddev 26.141, 0 failed
progress: 54.0 s, 453.2 tps, lat 17.644 ms stddev 15.753, 0 failed
progress: 60.0 s, 675.8 tps, lat 11.882 ms stddev 14.250, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31473
number of failed transactions: 0 (0.000%)
latency average = 15.248 ms 
latency stddev = 22.229 ms 
initial connection time = 17.837 ms
tps = 524.518721 (without initial connection time)

4. Текущие параметры:
postgres=# select name, setting, short_desc from pg_settings where name in (
'max_connections',
'shared_buffers',
'effective_cache_size',
'maintenance_work_mem',
'checkpoint_completion_target',
'wal_buffers',
'default_statistics_target',
'random_page_cost',
'effective_io_concurrency',
'work_mem',
'min_wal_size',
'max_wal_size');
             name             | setting |                                        short_desc
------------------------------+---------+------------------------------------------------------------------------------------------
 checkpoint_completion_target | 0.9     | Time spent flushing dirty buffers during checkpoint, as fraction of checkpoint interval.
 default_statistics_target    | 100     | Sets the default statistics target.
 effective_cache_size         | 524288  | Sets the planner's assumption about the total size of the data caches.
 effective_io_concurrency     | 1       | Number of simultaneous requests that can be handled efficiently by the disk subsystem.
 maintenance_work_mem         | 65536   | Sets the maximum memory to be used for maintenance operations.
 max_connections              | 100     | Sets the maximum number of concurrent connections.
 max_wal_size                 | 1024    | Sets the WAL size that triggers a checkpoint.
 min_wal_size                 | 80      | Sets the minimum size to shrink the WAL to.
 random_page_cost             | 4       | Sets the planner's estimate of the cost of a nonsequentially fetched disk page.
 shared_buffers               | 16384   | Sets the number of shared memory buffers used by the server.
 wal_buffers                  | 512     | Sets the number of disk-page buffers in shared memory for WAL.
 work_mem                     | 4096    | Sets the maximum memory to be used for query workspaces.
(12 rows)

5. Редактируем файл настроек согласно приложенному к заданию файлу:
otus@otus-db-pg-vm-10:~$ sudo nano /etc/postgresql/15/main/postgresql.conf
Параметры из файла:
*max_connections = 40;
*shared_buffers = 1GB
*effective_cache_size = 3GB
*maintenance_work_mem = 512MB
*checkpoint_completion_target = 0.9
*wal_buffers = 16MB
*default_statistics_target = 500
*random_page_cost = 4
*effective_io_concurrency = 2
*work_mem* = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

6. Перестартовываем постгрес

7. Выборка настроек после изменений:

postgres=# select name, setting, short_desc from pg_settings where name in (
'max_connections',
'shared_buffers',
'effective_cache_size',
'maintenance_work_mem',
'checkpoint_completion_target',
'wal_buffers',
'default_statistics_target',
'random_page_cost',
'effective_io_concurrency',
'work_mem',
'min_wal_size',
'max_wal_size');
             name             | setting |                                        short_desc
------------------------------+---------+------------------------------------------------------------------------------------------
 checkpoint_completion_target | 0.9     | Time spent flushing dirty buffers during checkpoint, as fraction of checkpoint interval.
 default_statistics_target    | 500     | Sets the default statistics target.
 effective_cache_size         | 393216  | Sets the planner's assumption about the total size of the data caches.
 effective_io_concurrency     | 2       | Number of simultaneous requests that can be handled efficiently by the disk subsystem.
 maintenance_work_mem         | 524288  | Sets the maximum memory to be used for maintenance operations.
 max_connections              | 40      | Sets the maximum number of concurrent connections.
 max_wal_size                 | 16384   | Sets the WAL size that triggers a checkpoint.
 min_wal_size                 | 4096    | Sets the minimum size to shrink the WAL to.
 random_page_cost             | 4       | Sets the planner's estimate of the cost of a nonsequentially fetched disk page.
 shared_buffers               | 131072  | Sets the number of shared memory buffers used by the server.
 wal_buffers                  | 2048    | Sets the number of disk-page buffers in shared memory for WAL.
 work_mem                     | 6553    | Sets the maximum memory to be used for query workspaces.


otus@otus-db-pg-vm-10:~$ sudo su - postgres
postgres@otus-db-pg-vm-10:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 419.2 tps, lat 19.000 ms stddev 13.477, 0 failed
progress: 12.0 s, 571.7 tps, lat 13.990 ms stddev 10.771, 0 failed
progress: 18.0 s, 428.0 tps, lat 18.668 ms stddev 15.726, 0 failed
progress: 24.0 s, 609.2 tps, lat 13.157 ms stddev 9.057, 0 failed
progress: 30.0 s, 602.3 tps, lat 13.278 ms stddev 10.178, 0 failed
progress: 36.0 s, 519.2 tps, lat 15.348 ms stddev 11.790, 0 failed
progress: 42.0 s, 498.5 tps, lat 16.104 ms stddev 13.717, 0 failed
progress: 48.0 s, 467.2 tps, lat 17.129 ms stddev 16.420, 0 failed
progress: 54.0 s, 511.2 tps, lat 15.648 ms stddev 11.649, 0 failed
progress: 60.0 s, 622.2 tps, lat 12.859 ms stddev 9.510, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 31499
number of failed transactions: 0 (0.000%)
latency average = 15.235 ms
latency stddev = 12.350 ms
initial connection time = 19.398 ms
tps = 524.974453 (without initial connection time)

После изменения настроек показатели существенно не изменились, но имеет смысл обратить внимание на измениемые параметры.

8. Создаем таблицу с текстовым полем и заполняем ее случайными или сгенерированными данным в размере 1млн строк:
postgres=# create table test as
select
  generate_series(1,1000000) as id,
  md5(random()::text)::char(10) as fio;
SELECT 1000000
postgres=# select count(*) from test;
  count
---------
 1000000
(1 row)

9. Запросом получаем размер таблицы + индексов и кол-во строк (запрос пригодится потом в работе)
postgres=# SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE C.relname = 'test'
ORDER BY pg_total_relation_size (C.oid) DESC;
 schemaname | relation | table | index | table_index | n_live_tup
------------+----------+-------+-------+-------------+------------
 public     | test     | 42 MB | 40 kB | 42 MB       |    1000000

10. В последнем поле запроса можно посмотреть время последнего автовакуума и количества мертвых строк:

postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_all_tables where relname = 'test';
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | test    |    1000000 |          0 | 2023-06-17 15:22:44.421974+00

11. Проапдейтил 5 раз:
update test set fio = fio || '1'; commit;
ERROR:  value too long for type character(10)
WARNING:  there is no transaction in progress
COMMIT
postgres=# update test set fio = substr(fio, 1, 9) || '1'; commit;
UPDATE 1000000
WARNING:  there is no transaction in progress
COMMIT
postgres=# update test set fio = substr(fio, 1, 9) || '2'; commit;
UPDATE 1000000
WARNING:  there is no transaction in progress
COMMIT
postgres=# update test set fio = substr(fio, 1, 9) || '3'; commit;
UPDATE 1000000
WARNING:  there is no transaction in progress
COMMIT
postgres=# update test set fio = substr(fio, 1, 9) || '4'; commit;
UPDATE 1000000
WARNING:  there is no transaction in progress
COMMIT
postgres=# update test set fio = substr(fio, 1, 9) || '5'; commit;
UPDATE 1000000
WARNING:  there is no transaction in progress
COMMIT

12. Кол-во мертвы строк и время автовакуума. Похоже, что автоваккум прошел перед последним адейтом:
postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_all_tables where relname = 'test';
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | test    |    1000000 |    1000000 | 2023-06-17 16:02:58.628338+00
(1 row)

13. А вот и дождались: 
postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_all_tables where relname = 'test';
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | test    |    1466762 |          0 | 2023-06-17 16:03:45.691335+00
(1 row)

14. Размер файла с таблицей:
postgres=# SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE C.relname = 'test'
ORDER BY pg_total_relation_size (C.oid) DESC
;
 schemaname | relation | table  | index | table_index | n_live_tup
------------+----------+--------+-------+-------------+------------
 public     | test     | 211 MB | 80 kB | 211 MB      |    1000000

15. Отключаем автовакуум для таблицы:
postgres=# ALTER TABLE test SET (autovacuum_enabled = off);
ALTER TABLE

16. Вносим изменения 10 раз аналогично п.11

17. Размеры таблицы:
postgres=# SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE C.relname = 'test'
ORDER BY pg_total_relation_size (C.oid) DESC
;
 schemaname | relation | table  | index  | table_index | n_live_tup
------------+----------+--------+--------+-------------+------------
 public     | test     | 464 MB | 144 kB | 464 MB      |    1000000
(1 row)

18. Рамер таблицы увеличился за счет отключения автовакуума и вследствии этого наличия мертвых строк:
postgres=# SELECT schemaname, relname, n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_all_tables where relname = 'test';
 schemaname | relname | n_live_tup | n_dead_tup |        last_autovacuum
------------+---------+------------+------------+-------------------------------
 public     | test    |    1000000 |    9995774 | 2023-06-17 16:09:46.304727+00