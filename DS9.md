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
