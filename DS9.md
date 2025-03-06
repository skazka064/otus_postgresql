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
postgres=# 
```
