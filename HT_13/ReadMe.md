-- ДЗ тема: триггеры, поддержка заполнения витрин

DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);

INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL,
    unique(good_name)
);

-- Создать триггер (на таблице sales) для поддержки.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.

РЕШЕНИЕ:
1. Для данной задачи необязательно поднимать образ постгреса на ВМ. Использовался кластер на локальной машине.
2. Выполнены скрипты из задания - созданы таблицы и наполнены данными:

select * from goods;

goods_id|good_name               |good_price  |
--------+------------------------+------------+
       1|Спички хозайственные    |        0.50|
       2|Автомобиль Ferrari FXX K|185000000.01|

select * from sales;

sales_id|good_id|sales_time                   |sales_qty|
--------+-------+-----------------------------+---------+
       1|      1|2023-08-02 18:37:31.503 +0300|       10|
       2|      1|2023-08-02 18:37:31.503 +0300|        1|
       3|      1|2023-08-02 18:37:31.503 +0300|      120|
       4|      2|2023-08-02 18:37:31.503 +0300|        1|

3. При добавлении записи мы должны добавить сумму продажи, при изменении добавить/убрать разницу, при удалении - убрать сумму продажи из таблицы-витрины.
   Создаю триггерную функцию на удаление, изменение, добавление:

create or replace function sales_trg1() returns trigger as $$
declare
  i_good_id integer;
  i_sales_qty integer;
begin
  case TG_OP
    when 'INSERT' then
      i_good_id = new.good_id;
      i_sales_qty = new.sales_qty;
    when 'UPDATE' then
      i_good_id = old.good_id;
      i_sales_qty = new.sales_qty - old.sales_qty;
    when 'DELETE' then
      i_good_id = old.good_id;
      i_sales_qty = - old.sales_qty;
  end case;
  insert into good_sum_mart(good_name, sum_sale) select good_name, good_price * i_sales_qty from goods where goods_id = i_good_id
  on conflict (good_name) do update set sum_sale = good_sum_mart.sum_sale + i_sales_qty where good_sum_mart.good_name = i_good_id;
  return null;
end;
$$ language plpgsql;

Чтобы можно было использовать конструкцию  "on conflict" при вставке - необходимо добавить "unique(good_name)" при создании good_sum_mart;

4. Создаю триггер:

create trigger sales_trg1
after insert or update or delete on sales
for each row execute function sales_trg1();

5. Удаляю данные из таблицы sales и good_sum_mart и заново добавляю заново данные в таблицу sales.

Результат:
select * from good_sum_mart;
good_name               |sum_sale    |
------------------------+------------+
Спички хозайственные    |       65.50|
Автомобиль Ferrari FXX K|185000000.01|

6. Удаляю данные из таблицы sales

Результат:
select * from good_sum_mart;
good_name               |sum_sale|
------------------------+--------+
Спички хозайственные    |    0.00|
Автомобиль Ferrari FXX K|    0.00|

7. Добавлю одну строку в таблицу sales:

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10)

Результат:
select * from good_sum_mart;

good_name               |sum_sale|
------------------------+--------+
Автомобиль Ferrari FXX K|    0.00|
Спички хозайственные    |    5.00|

8. Апдейчу данные в таблицу sales:

update sales set sales_qty = 100 where good_id = 1;

Результат:
select * from good_sum_mart;

good_name               |sum_sale|
------------------------+--------+
Автомобиль Ferrari FXX K|    0.00|
Спички хозайственные    |   50.00|

ВЫВОД: Созданный триггер работает корректно при INSERT, UPDATE, DELETE.

Возможны ситуации, когда триггер может не отработать и данные в таблице-витрине не обновятся. 
Плюс - что таблица наполнена актуальными данными и выборка происходит быстрее, чем при использовании отчета (селекта из 2-х таблиц).
Опять же при больших объемах данных ее можно секционировать и увеличить скорость выборки.