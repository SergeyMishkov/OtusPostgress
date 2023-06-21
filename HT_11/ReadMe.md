0. Выбран второй вариант выполнения ДЗ.  
1. Создаем таблицы:  

Клиенты:
create table clients (id serial, first_name text, second_name text, address text); 
insert into clients
select generate_series(1,6),
       md5(random()::text)::char(10) as first_name,
       md5(random()::text)::char(10) as second_name,
       md5(random()::text)::char(30) as address;
      
select * from clients c   

id|first_name|second_name|address                       |  
--+----------+-----------+------------------------------+  
 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|  
 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|  
 3|03bbdb8c3e|d1f40ae891 |f1bc4e9c5d46bd64c656a4a69d99e8|  
 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|  
 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|  
 6|a1137f685b|2dd44fef3e |a2783839fec607d0ba01a07f74091d|  

Договора:

create table contracts(id serial, cont_number text, client numeric, product numeric);
insert into contracts(cont_number, client, product) values('Договор 1', 1, 1001); 
insert into contracts(cont_number, client, product) values('Договор 2', 2, 1001); 
insert into contracts(cont_number, client, product) values('Договор 3', 4, 1004); 
insert into contracts(cont_number, client, product) values('Договор 4', 5, 1005); 
insert into contracts(cont_number, client, product) values('Договор 5', 5, 1001); 

select * from contracts c 

id|cont_number|client|product|
--+-----------+------+-------+
 1|Договор 1  |     1|   1001|
 2|Договор 2  |     2|   1001|
 3|Договор 3  |     4|   1004|
 4|Договор 4  |     5|   1005|
 5|Договор 5  |     5|   1001|

Счета:

create table accounts(id_contract numeric, id_type numeric, amount numeric);

insert into accounts(id_contract, id_type, amount) values(1, 101, 100);
insert into accounts(id_contract, id_type, amount) values(1, 102, 90);
insert into accounts(id_contract, id_type, amount) values(1, 103, 0);
insert into accounts(id_contract, id_type, amount) values(2, 101, 100);
insert into accounts(id_contract, id_type, amount) values(3, 102, 99);
insert into accounts(id_contract, id_type, amount) values(5, 101, 500);
insert into accounts(id_contract, id_type, amount) values(5, 102, 50);
insert into accounts(id_contract, id_type, amount) values(5, 103, 55);

select * from accounts;

id_contract|id_type|amount|
-----------+-------+------+
          1|    101|   100|
          1|    102|    90|
          1|    103|     0|
          2|    101|   100|
          3|    102|    99|
          5|    101|   500|
          5|    102|    50|

2. Прямое соединение
   Клиенты, имеющие договора - выбираются все записи из таблицы clients имеющих ссылку на таблицу contracts (distinct убирает дубли).

select distinct b.*
from  contracts a 
inner join clients b on a.client = b.id;

Результат выполнения запроса:

id|first_name|second_name|address                       |
--+----------+-----------+------------------------------+
 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|
 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|
 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|
 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|
 
 3. Левое соединение (на практике правое соединение за все время встречал только раз и тот можно было переписать используя только левое)
    в результаты выводятся договора с остатками на счетах опредедленного типа (101-103)

 select c.*,
        a101.amount amount_101,
        a102.amount amount_102,
        a103.amount amount_103
 from contracts c
 left join accounts a101 on c.id = a101.id_contract and a101.id_type = 101
 left join accounts a102 on c.id = a102.id_contract and a102.id_type = 102
 left join accounts a103 on c.id = a103.id_contract and a103.id_type = 103;

 Результат выполнения запроса:

id|cont_number|client|product|amount_101|amount_102|amount_103|
--+-----------+------+-------+----------+----------+----------+
 1|Договор 1  |     1|   1001|       100|        90|         0|
 2|Договор 2  |     2|   1001|       100|          |          |
 3|Договор 3  |     4|   1004|          |        99|          |
 4|Договор 4  |     5|   1005|          |          |          |
 5|Договор 5  |     5|   1001|       500|        50|        55|

4.  Перекрестное соединение (перемножение таблиц)
    возвратит количество строк 5 * 6 = 30 строк

select *
from  contracts a cross join clients b;

Результат выполнения запроса:

id|cont_number|client|product|id|first_name|second_name|address                       |
--+-----------+------+-------+--+----------+-----------+------------------------------+
 1|Договор 1  |     1|   1001| 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|
 1|Договор 1  |     1|   1001| 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|
 1|Договор 1  |     1|   1001| 3|03bbdb8c3e|d1f40ae891 |f1bc4e9c5d46bd64c656a4a69d99e8|
 1|Договор 1  |     1|   1001| 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|
 1|Договор 1  |     1|   1001| 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|
 1|Договор 1  |     1|   1001| 6|a1137f685b|2dd44fef3e |a2783839fec607d0ba01a07f74091d|
 2|Договор 2  |     2|   1001| 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|
 2|Договор 2  |     2|   1001| 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|
 2|Договор 2  |     2|   1001| 3|03bbdb8c3e|d1f40ae891 |f1bc4e9c5d46bd64c656a4a69d99e8|
 2|Договор 2  |     2|   1001| 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|
 2|Договор 2  |     2|   1001| 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|
 2|Договор 2  |     2|   1001| 6|a1137f685b|2dd44fef3e |a2783839fec607d0ba01a07f74091d|
 3|Договор 3  |     4|   1004| 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|
 3|Договор 3  |     4|   1004| 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|
 3|Договор 3  |     4|   1004| 3|03bbdb8c3e|d1f40ae891 |f1bc4e9c5d46bd64c656a4a69d99e8|
 3|Договор 3  |     4|   1004| 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|
 3|Договор 3  |     4|   1004| 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|
 3|Договор 3  |     4|   1004| 6|a1137f685b|2dd44fef3e |a2783839fec607d0ba01a07f74091d|
 4|Договор 4  |     5|   1005| 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|
 4|Договор 4  |     5|   1005| 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|
 4|Договор 4  |     5|   1005| 3|03bbdb8c3e|d1f40ae891 |f1bc4e9c5d46bd64c656a4a69d99e8|
 4|Договор 4  |     5|   1005| 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|
 4|Договор 4  |     5|   1005| 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|
 4|Договор 4  |     5|   1005| 6|a1137f685b|2dd44fef3e |a2783839fec607d0ba01a07f74091d|
 5|Договор 5  |     5|   1001| 1|6b4f2ac9f7|1291c6bbf3 |1587bbb03023e3b354aeaf55ea08f2|
 5|Договор 5  |     5|   1001| 2|33c99ae360|0ed54d6416 |b798009cb7b6c9bccf0944b0e67963|
 5|Договор 5  |     5|   1001| 3|03bbdb8c3e|d1f40ae891 |f1bc4e9c5d46bd64c656a4a69d99e8|
 5|Договор 5  |     5|   1001| 4|cae2c0f137|a163117223 |9f76b892f6d56aae4cd8e903be1f6c|
 5|Договор 5  |     5|   1001| 5|5670ab857e|08a710a7a3 |dc64f68b6f34b5825e05fc56f4585b|
 5|Договор 5  |     5|   1001| 6|a1137f685b|2dd44fef3e |a2783839fec607d0ba01a07f74091d|
 
5. Полное соединение
   выбираются строки из нескольких таблиц (подзапросов), соединенных по ключу, как присутствующие в каждой из таблиц, так отсутствующие

 select *
 from (select a.id_contract, a.amount from accounts a where a.id_type = 101) a101
 full join (select b.id_contract, b.amount from accounts b where b.id_type = 102) a102 
 on a101.id_contract = a102.id_contract;

 Результат выполнения запроса:

id_contract|amount|id_contract|amount|
-----------+------+-----------+------+
          1|   100|          1|    90|
          2|   100|           |      |
          5|   500|          5|    50|
           |      |          3|    99|
           
6. Пример запроса с несколькими соединениями
   выводится сумма остатков на счетах по клиентам в разрезе договоров 

select a.first_name, a.second_name, b.cont_number, sum(c.amount) 
from  clients a 
inner join contracts b on a.id = b.client 
left join accounts c on b.id = c.id_contract
group by a.first_name, a.second_name, b.cont_number
order by 1, 3;

Результат выполнения запроса:

first_name|second_name|cont_number|sum|
----------+-----------+-----------+---+
33c99ae360|0ed54d6416 |Договор 2  |100|
5670ab857e|08a710a7a3 |Договор 4  |   |
5670ab857e|08a710a7a3 |Договор 5  |605|
6b4f2ac9f7|1291c6bbf3 |Договор 1  |190|
cae2c0f137|a163117223 |Договор 3  | 99|