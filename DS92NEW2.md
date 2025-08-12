### В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).
### Есть запрос для генерации отчета – сумма продаж по каждому товару.
### БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.
### Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

```sql
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);

INSERT INTO goods (goods_id, good_name, good_price) VALUES (1, 'Спички хозайственные', .50),(2, 'Автомобиль Ferrari FXX K', 185000000.01);

CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```
# INSERT
```sql
CREATE OR REPLACE FUNCTION t_insert()
RETURNS trigger 
AS
$tr$
BEGIN
	IF TG_OP = 'INSERT' THEN
		IF (SELECT good_name FROM goods WHERE goods_id=NEW.good_id)=(SELECT good_name FROM good_sum_mart  WHERE good_name=(SELECT good_name FROM goods WHERE goods_id=NEW.good_id )) THEN
		UPDATE good_sum_mart SET sum_sale=(SELECT sum_sale FROM good_sum_mart WHERE good_name=(SELECT good_name FROM goods WHERE goods_id=NEW.good_id))+(NEW.sales_qty*(SELECT good_price FROM goods WHERE goods_id=NEW.good_id)) WHERE good_name=(SELECT good_name FROM goods WHERE goods_id=NEW.good_id);
	    ELSE
		
		INSERT INTO good_sum_mart VALUES ((SELECT good_name FROM goods WHERE goods_id=NEW.good_id),((SELECT good_price FROM goods WHERE goods_id=NEW.good_id)*NEW.sales_qty));
		END IF;
	END IF;
	RAISE NOTICE 'NEW.good_id=% NEW.sales_qty=% ', NEW.good_id, NEW.sales_qty;
	RETURN NEW;
END
$tr$
LANGUAGE plpgsql

CREATE TRIGGER t_ins
AFTER INSERT ON sales
FOR EACH ROW EXECUTE FUNCTION t_insert();

```
# UPDATE
```sql
CREATE OR REPLACE FUNCTION t_update()
RETURNS trigger
AS
$tr$
BEGIN
IF TG_OP = 'UPDATE' THEN
UPDATE good_sum_mart SET sum_sale=NEW.sales_qty*(SELECT good_price FROM goods WHERE goods_id=NEW.good_id) WHERE (sum_sale=OLD.sales_qty*(SELECT good_price FROM goods WHERE goods_id=NEW.good_id)) AND (good_name=(SELECT good_name FROM goods WHERE goods_id=NEW.good_id));
END IF;
RAISE NOTICE 'NEW%, OLD%',NEW,OLD;
RETURN NEW;
END
$tr$
LANGUAGE plpgsql;

CREATE TRIGGER t_upd 
AFTER UPDATE ON sales
FOR EACH ROW EXECUTE FUNCTION t_update();

```
