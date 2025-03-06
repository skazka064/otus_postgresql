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
postgres=# EXPLAIN
postgres-# SELECT * FROM documents
postgres-#     WHERE contents like '%document%';
                         QUERY PLAN                         
------------------------------------------------------------
 Seq Scan on documents  (cost=0.00..14.25 rows=1 width=210)
   Filter: (contents ~~ '%document%'::text)
(2 rows)
```
#### Теперь добавим GIN индекс на текст документа
```sql
postgres=# CREATE INDEX
postgres-#     idx_documents_contents
postgres-#     ON documents
postgres-#     USING GIN(to_tsvector('english', contents));
CREATE INDEX

```
```sql
postgres=# EXPLAIN SELECT * FROM documents
    WHERE to_tsvector('english', contents) @@ 'document';
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Seq Scan on documents  (cost=0.00..2.84 rows=1 width=210)
   Filter: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
(2 rows)
```
#### Стоимость запроса с индексом GIN уменьшилась в семь раз
#### Проверяем результат (с отключенным последовательным сканированием):
```sql
postgres=# SET enable_seqscan = OFF;
SET
postgres=# EXPLAIN
postgres-# SELECT * FROM documents
postgres-#     WHERE to_tsvector('english', contents) @@ 'document';
                                          QUERY PLAN                                          
----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on documents  (cost=8.00..12.26 rows=1 width=210)
   Recheck Cond: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
   ->  Bitmap Index Scan on idx_documents_contents  (cost=0.00..8.00 rows=1 width=0)
         Index Cond: (to_tsvector('english'::regconfig, contents) @@ '''document'''::tsquery)
