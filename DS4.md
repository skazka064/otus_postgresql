# BEGIN
## создайте новую базу данных testdb

```sql
postgres=# create database testdb;
CREATE DATABASE
postgres=#
```
## зайдите в созданную базу данных под пользователем postgres
```sql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```
## создайте новую схему testnm
```sql
testdb=# create schema testnm;
CREATE SCHEMA
testdb=#
```
## создайте новую таблицу t1 с одной колонкой c1 типа integer
```sql
testdb=# create table t1 (c1 integer);
CREATE TABLE
```
## вставьте строку со значением c1=1
```sql
testdb=# insert into t1 values (1);
INSERT 0 1
```
### создайте новую роль readonly
```sql
testdb=# create role readonly;
CREATE ROLE
```
### дайте новой роли право на подключение к базе данных testdb
```sql
testdb=# grant connect on database testdb to readonly ;
GRANT
```

