1. Развернул ВМ в Яндекс-облаке.
2. Подключился командой:
ssh -i ~/.ssh/yc_key otus@158.160.12.147
3. Установил постгрес 15 командой:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
4. Зашел в созданный кластер под пользователем postgres:
sudo -u postgres psql
5. Создаем БД otus.
postgres=# create database otus;
CREATE DATABASE
6. Переходим в БД otus.
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
6. Создаем схему testscheme.
otus=# CREATE SCHEMA testscheme;
CREATE SCHEMA
7. В схеме testscheme создаем таблицу workers c автосгенерированными 100 записями:
otus=# create table testscheme.workers as
select
  generate_series(1,100) as id,
  md5(random()::text)::char(10) as fio;
SELECT 100
8. Проверяем:
otus=# select count(*) from testscheme.workers;
 count
-------
   100
(1 row)
9. Текущая папка:
otus@otus-vm-db-pg-net-1:~$ pwd
/home/otus
10. Создаем папку для бэкапов:
postgres@otus-vm-db-pg-net-1:~$ mkdir backups
11. Проверяем:
postgres@otus-vm-db-pg-net-1:~/backups$ pwd
/var/lib/postgresql/backups
12. Запускаем логический бэкап данных таблицы:
otus=# \copy testscheme.workers to '/var/lib/postgresql/backups/workers.sql'
COPY 100
13. Удаляем данные из таблицы и проверяем:
otus=# delete from testscheme.workers;
DELETE 100
otus=# select count(*) from testscheme.workers;
 count
-------
     0
(1 row)

14. Запускаем восстановление из бэкапа данных таблицы:
otus=# \copy testscheme.workers from '/var/lib/postgresql/backups/workers.sql'
COPY 100

15. Проверяем - данные восстановлены:
otus=# select count(*) from testscheme.workers;
 count
-------
   100
(1 row)

16. Список БД в кластере:
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(4 rows)

17. Создаем копию БД otus:
postgres@otus-vm-db-pg-net-1:~$ pg_dump -d otus --create -U postgres -Fc > /var/lib/postgresql/backups/arh.gz

18. Файл создался:
postgres@otus-vm-db-pg-net-1:~$ ls -l /var/lib/postgresql/backups/
total 8
-rw-rw-r-- 1 postgres postgres 2298 Jun 12 12:37 arh.gz
-rw-rw-r-- 1 postgres postgres 1392 Jun 12 11:50 workers.sql

19. Удаляем БД otus
postgres@otus-vm-db-pg-net-1:~$ echo "DROP DATABASE otus;" | psql -U postgres
DROP DATABASE

20. Список БД:
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(3 rows)

21. Создаем пустую БД otus:
postgres@otus-vm-db-pg-net-1:~$ echo "CREATE DATABASE otus;" | psql -U postgres
CREATE DATABASE

22. Восстанавливвем из резервной копии:
postgres@otus-vm-db-pg-net-1:~$  pg_restore -d otus -U postgres /var/lib/postgresql/backups/arh.gz

23. Проверяем, что база восстановилась - есть ранее созданная таблица:
otus=# select count(*) from testscheme.workers;
 count
-------
   100
(1 row)