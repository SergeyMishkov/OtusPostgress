1. Развернул ВМ в Яндекс-облаке.
2. Подключился командой:
ssh -i ~/.ssh/yc_key otus@158.160.18.224
3. Установил Docker Engine командой:
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER
4. Создаем контейнер установкой posgres 15:
sudo docker network create pg-net
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
5. Запускаем отдельный контейнер с клиентом в общей сети с БД: 
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
6. Список доступных контейнеров

otus@otus-db-pg-vm-1:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS             PORTS                                       NAMES
e946d5deb4a2   postgres:15   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

7. Подключился из контейнера с клиентом к контейнеру с сервером

otus@otus-db-pg-vm-1:~$ psql -h localhost -U postgres -d postgres
Password for user postgres:
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(3 rows)

8. Создал таблицу и добавил записи:

postgres=# create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1

Таблица добавилсь:
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

9. Подключился с ноутбука
otus@otus-db-pg-vm-1:~$ psql -p 5432 -U postgres -h 158.160.18.224 -d postgres -W
Password:
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
Type "help" for help.

10. Останавливаем удаляем контейнер, проверяем что он удалился:
otus@otus-db-pg-vm-1:~$ sudo docker stop pg-server
pg-server
otus@otus-db-pg-vm-1:~$ sudo docker rm pg-server
pg-server
otus@otus-db-pg-vm-1:~$ sudo docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
otus@otus-db-pg-vm-1:~$

11. Пересоздали контейнер и проверили что он существует:

otus@otus-db-pg-vm-1:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                                       NAMES
1f1374147d90   postgres:15   "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

12. Подключились к базе - данные остались

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
