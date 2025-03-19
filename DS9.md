### Создать индекс к какой-либо из таблиц вашей БД
```sql
postgres=# create index idx_userid on users_index (userid);
CREATE INDEX
postgres=# 
postgres=# \d users_index 
              Table "public.users_index"
    Column    | Type | Collation | Nullable | Default 
--------------+------+-----------+----------+---------
 userid       | text |           |          | 
 username     | text |           |          | 
 email        | text |           |          | 
 avatar       | text |           |          | 
 password     | text |           |          | 
 birthdate    | text |           |          | 
 registeredat | text |           |          | 
Indexes:
    "idx_userid" btree (userid)
```
### Прислать текстом результат команды explain, в которой используется данный индекс
```sql
postgres=# explain select * from users_index where userid='75b4ba5f-f070-4404-b68a-8900cd222506';
                                   QUERY PLAN                                   
--------------------------------------------------------------------------------
 Index Scan using idx_userid on users_index  (cost=0.28..8.29 rows=1 width=213)
   Index Cond: (userid = '75b4ba5f-f070-4404-b68a-8900cd222506'::text)
(2 rows)

```
### Реализовать индекс для полнотекстового поиска
#### Для этого создаём и заполняем таблицу с документами
```sql
postgres=# CREATE TABLE documents (
postgres(#     title    varchar(64),
postgres(#     metadata jsonb,
postgres(#     contents text
postgres(# );
CREATE TABLE
postgres=# INSERT INTO documents
postgres-#     (title, metadata, contents)
postgres-# VALUES
postgres-#     ( 'Document 1',
postgres(#       '{"author": "John",  "tags": ["legal", "real estate"]}',
postgres(#       'This is a legal document about real estate.' ),
postgres-#     ( 'Document 2',
postgres(#       '{"author": "Jane",  "tags": ["finance", "legal"]}',
postgres(#       'Financial statements should be verified.' ),
postgres-#     ( 'Document 3',
postgres(#       '{"author": "Paul",  "tags": ["health", "nutrition"]}',
postgres(#       'Regular exercise promotes better health.' ),
postgres-#     ( 'Document 4',
postgres(#       '{"author": "Alice", "tags": ["travel", "adventure"]}',
postgres(#       'Mountaineering requires careful preparation.' ),
postgres-#     ( 'Document 5',
postgres(#       '{"author": "Bob",   "tags": ["legal", "contracts"]}',
postgres(#       'Contracts are binding legal documents.' ),
postgres-#     ( 'Document 6',
postgres(#        '{"author": "Eve",  "tags": ["legal", "family law"]}',
postgres(#        'Family law addresses diverse issues.' ),
postgres-#     ( 'Document 7',
postgres(#       '{"author": "John",  "tags": ["technology", "innovation"]}',
postgres(#       'Tech innovations are changing the world.' );
INSERT 0 7
postgres=# SELECT * FROM documents;
   title    |                         metadata                         |                   contents                   
------------+----------------------------------------------------------+----------------------------------------------
 Document 1 | {"tags": ["legal", "real estate"], "author": "John"}     | This is a legal document about real estate.
 Document 2 | {"tags": ["finance", "legal"], "author": "Jane"}         | Financial statements should be verified.
 Document 3 | {"tags": ["health", "nutrition"], "author": "Paul"}      | Regular exercise promotes better health.
 Document 4 | {"tags": ["travel", "adventure"], "author": "Alice"}     | Mountaineering requires careful preparation.
 Document 5 | {"tags": ["legal", "contracts"], "author": "Bob"}        | Contracts are binding legal documents.
 Document 6 | {"tags": ["legal", "family law"], "author": "Eve"}       | Family law addresses diverse issues.
 Document 7 | {"tags": ["technology", "innovation"], "author": "John"} | Tech innovations are changing the world.
(7 rows)
postgres=# SELECT * FROM documents
postgres-#     WHERE metadata @> '{"author": "John"}';
   title    |                         metadata                         |                  contents                   
------------+----------------------------------------------------------+---------------------------------------------
 Document 1 | {"tags": ["legal", "real estate"], "author": "John"}     | This is a legal document about real estate.
 Document 7 | {"tags": ["technology", "innovation"], "author": "John"} | Tech innovations are changing the world.
(2 rows)

```
#### Посмотрим поиск по условию с LIKE
```sql
postgres=# EXPLAIN ANALYZE
SELECT * FROM documents
    WHERE contents like '%document%';
                                             QUERY PLAN                                              
-----------------------------------------------------------------------------------------------------
 Seq Scan on documents  (cost=0.00..1.09 rows=1 width=116) (actual time=0.009..0.012 rows=2 loops=1)
   Filter: (contents ~~ '%document%'::text)
   Rows Removed by Filter: 5
 Planning Time: 0.111 ms
 Execution Time: 0.027 ms
(5 rows)

```
#### Теперь добавим GIN индекс на текст документа
```sql
postgres=# CREATE INDEX ON documents USING GIN(to_tsvector('english', contents));
CREATE INDEX

postgres=# ANALYZE documents ;
ANALYZE
postgres=# EXPLAIN ANALYZE SELECT * FROM documents
    WHERE to_tsvector('english', contents) @@ 'document';
                                             QUERY PLAN                                              
-----------------------------------------------------------------------------------------------------
 Seq Scan on documents  (cost=0.00..2.84 rows=2 width=116) (actual time=0.050..0.141 rows=2 loops=1)
   Filter: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
   Rows Removed by Filter: 5
 Planning Time: 0.078 ms
 Execution Time: 0.164 ms
(5 rows)

```
#### В обоих случаях использовался seqscan и с применением индекса время запроса увеличилось
#### Проверяем то же самое с отключенным последовательным сканированием
```sql
postgres=# SET enable_seqscan = OFF;
SET
postgres=# EXPLAIN ANALYZE
SELECT * FROM documents
    WHERE to_tsvector('english', contents) @@ 'document';
                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on documents  (cost=8.00..12.26 rows=1 width=210) (actual time=0.041..0.043 rows=2 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_documents_contents  (cost=0.00..8.00 rows=1 width=0) (actual time=0.030..0.030 rows=2 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
 Planning Time: 0.089 ms
 Execution Time: 0.139 ms
(7 rows)

```
#### Мы здесь видим, что применился Bitmap Heap Scan
#### и посмотрим select с LIKE с отключенным seqscan
```sql
postgres=# EXPLAIN ANALYZE
SELECT * FROM documents
    WHERE contents like '%document%';
                                                         QUERY PLAN                                                          
-----------------------------------------------------------------------------------------------------------------------------
 Seq Scan on documents  (cost=10000000000.00..10000000001.09 rows=1 width=210) (actual time=177.770..177.776 rows=2 loops=1)
   Filter: (contents ~~ '%document%'::text)
   Rows Removed by Filter: 5
 Planning Time: 0.079 ms
 JIT:
   Functions: 2
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 0.714 ms, Inlining 82.014 ms, Optimization 42.091 ms, Emission 52.133 ms, Total 176.953 ms
 Execution Time: 178.538 ms
(9 rows)

```
### Реализовать индекс на часть таблицы или индекс на поле с функцией
```sql
postgres=# create table cities(
postgres(# code CHAR(13) primary key,
postgres(# name VARCHAR(255) not null
postgres(# );
CREATE TABLE
postgres=# WITH random_data AS (
postgres(#     SELECT
postgres(#     num,
postgres(#     random() AS rand1,
postgres(#     random() AS rand2,
postgres(#     random() AS rand3
postgres(#     FROM generate_series(1, 100000) AS s(num)
postgres(# )
postgres-# INSERT INTO cities
postgres-#     (code, name)
postgres-# SELECT
postgres-#     concat(
postgres(#         (random_data.rand1 * 10)::integer % 10,
postgres(#         (random_data.rand2 * 10)::integer % 10,
postgres(#         lpad(random_data.num::text, 11, '0')
postgres(#     ),
postgres-#     chr((32 + random_data.rand3 * 94)::integer)
postgres-#     FROM random_data
postgres-#     ORDER BY random();
INSERT 0 100000
postgres=# SELECT * FROM cities;
postgres=# EXPLAIN
postgres-# SELECT * FROM cities
postgres-#     WHERE SUBSTRING(code, 1, 2) = '50';
                         QUERY PLAN                         
------------------------------------------------------------
 Seq Scan on cities  (cost=0.00..2291.00 rows=500 width=16)
   Filter: ("substring"((code)::text, 1, 2) = '50'::text)
(2 rows)

postgres=# SELECT count(*) FROM cities
    WHERE SUBSTRING(code, 1, 2) = '50';
 count 
-------
  1022
(1 row)

postgres=# CREATE INDEX
postgres-#     func_idx_cities_region
postgres-#     ON cities ((SUBSTRING(code, 1, 2)));
CREATE INDEX
postgres=# ANALYZE cities;
ANALYZE
postgres=# EXPLAIN
postgres-# SELECT * FROM cities
postgres-#     WHERE SUBSTRING(code, 1, 2) = '50';
                                       QUERY PLAN                                       
----------------------------------------------------------------------------------------
 Bitmap Heap Scan on cities  (cost=20.09..589.21 rows=990 width=16)
   Recheck Cond: ("substring"((code)::text, 1, 2) = '50'::text)
   ->  Bitmap Index Scan on func_idx_cities_region  (cost=0.00..19.84 rows=990 width=0)
         Index Cond: ("substring"((code)::text, 1, 2) = '50'::text)
(4 rows)

```
#### вывод: без индекса цена получилась cost=0.00..2291.00 ; а с индексом cost=20.09..589.21 : цена уменьшилась в четыре раза
