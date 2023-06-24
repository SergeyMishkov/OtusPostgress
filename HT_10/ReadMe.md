1. Создал ВМ №1 в ЯО:  
   Имя: otus-db-pg-vm-1  
   IP внешний: 158.160.73.62  
   IP внутренний: 10.129.0.20  
2. Подключился командой $ ssh -i ~/.ssh/yc_key otus@158.160.73.62  
3. Установил postgres:  
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' &&   wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15  
4. Проверил создание кластера:  
otus@otus-db-pg-vm-1:~$ pg_lsclusters  
Ver Cluster Port Status Owner    Data directory              Log file  
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log  
5. Переходим в оболочку пострегрес:  
otus@otus-db-pg-vm-1:~$ sudo -u postgres psql  
6. Создал ВМ №2 в ЯО:  
   Имя: otus-db-pg-vm-2  
   IP внешний: 158.160.74.250  
   IP внутренний: 10.129.0.5    
7. Подключился командой $ ssh -i ~/.ssh/yc_key otus@158.160.74.250  
8. Установил postgres:  
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15  
9. Проверил создание кластера:  
otus@otus-db-pg-vm-2:~$ pg_lsclusters  
Ver Cluster Port Status Owner    Data directory              Log file  
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log  
10. Переходим в оболочку пострегрес:  
otus@otus-db-pg-vm-2:~$ sudo -u postgres psql  
11. Создаем БД otus в обеих ВМ и переходим в них:  
postgres=# create database otus;  
CREATE DATABASE  
postgres=# \c otus  
You are now connected to database "otus" as user "postgres".  
12. Создал две тестовые таблицы в базе на ВМ №1:  
otus=# create table test1(id serial, first_name text, second_name text);  
create table test2(id serial, first_name text, second_name text);  
CREATE TABLE  
CREATE TABLE  
13. Проверяю wal_level:  
otus=# show wal_level;  
 wal_level  
-----------  
 replica  
(1 row)  
14. Установили пароль для postgres:  
otus@otus-vm-db-pg-net-1:~$ sudo su - postgres  
postgres@otus-vm-db-pg-net-1:~$ psql -c "ALTER ROLE postgres PASSWORD 'pwd123'"  
ALTER ROLE  
15. Редактируем файл pg_hba.conf на ВМ1  
postgres@otus-vm-db-pg-net-1:~$ nano /etc/postgresql/15/main/pg_hba.conf  
добавляем строку:  
host    replication     all             10.129.0.5/32          scram-sha-256  
16. Редактируем файл   
nano /etc/postgresql/15/main/postgresql.conf  
listen_addresses = '*'  
wal_level = hot_standby  
archive_mode = on  
archive_command = 'cd .'  
max_wal_senders = 8  
hot_standby = on  
17. Перестартовываем постгрес на ВМ1:  
otus@otus-db-pg-vm-1:~$ sudo service postgresql restart  
18. Останавливаем ВМ2:  
otus@otus-db-pg-vm-2:~$ sudo service postgresql stop  
19. Для ВМ2 применяем настройки аналогичные   
20. Далее переходим в каталог с базой данных:    
cd /var/lib/postgresql/12/  
21. Удалим каталог с дефолтной БД и снова его создадим, но уже пустой:  
postgres@otus-db-pg-vm-2:~/15$ rm -rf main; mkdir main; chmod go-rwx main  
22. Восстанавливаем кластер на ВМ2 из ВМ1.  
postgres@otus-db-pg-vm-2:~/15$ pg_basebackup -P -R -X stream -c fast -h 10.129.0.20 -U postgres -D ./main  
Password:  
30590/30590 kB (100%), 1/1 tablespace  
23. Стартуем сервис на ВМ2:  
otus@otus-db-pg-vm-2:~$ sudo service postgresql start  
24. Добавляем запись в базу otus на ВМ1:  
otus=# insert into test1(first_name, second_name) values ('12345','56789');  
INSERT 0 1  
otus=# select * from test1;  
 id | first_name | second_name  
----+------------+-------------  
  1 | 12345      | 56789  
(1 row)  
25. Смотрим, что запись добавлена в базу otus на ВМ2:  
otus=# select * from test1;  
 id | first_name | second_name  
----+------------+-------------  
  1 | 12345      | 56789  
(1 row)  
Репликация работает!  
26. При попытке добавить запись в таблице на ВМ2 (на реплике):  
otus=# insert into test1(first_name, second_name) values ('12345','56789');  
ERROR:  cannot execute INSERT in a read-only transaction  