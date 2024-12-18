# BEGIN
### создайте новую базу данных testdb

```sql
postgres=# create database testdb;
CREATE DATABASE
postgres=#
```
### зайдите в созданную базу данных под пользователем postgres
```sql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```
### создайте новую схему testnm
```sql
testdb=# create schema testnm;
CREATE SCHEMA
testdb=#
```
### создайте новую таблицу t1 с одной колонкой c1 типа integer
```sql
testdb=# create table t1 (c1 integer);
CREATE TABLE
```
### вставьте строку со значением c1=1
```sql
testdb=# insert into t1 values (1);
INSERT 0 1
```
### создайте новую роль readonly
```sql
testdb=# create role readonly;
CREATE ROLE
testdb=# \du
                             List of roles
 Role name |                         Attributes
-----------+------------------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
 readonly  | Cannot login
```
### дайте новой роли право на подключение к базе данных testdb
```sql
testdb=# grant connect on database testdb to readonly ;
GRANT
```
### дайте новой роли право на использование схемы testnm
```sql
testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=#
```
### дайте новой роли право на select для всех таблиц схемы testnm
```sql
testdb=# grant select on all tables in schema testnm to readonly
testdb-# ;
GRANT
testdb=#
```
### создайте пользователя testread с паролем test123
```sql
testdb=# create user testread with password 'test123';
CREATE ROLE
testdb=#
```
### дайте роль readonly пользователю testread
```sql
testdb=# grant readonly to testread ;
GRANT ROLE
testdb=#
```
### зайдите под пользователем testread в базу данных testdb
```sql
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
testdb=# select * from t1;
 c1
----
  1
(1 row)

testdb=# \q

```
### такое происходит потому что метод аутентификации для локальных пользовалелей выставлен peer
### это означает что, получая имя пользователя ОС клиента из ядра используется в качестве разрешенного имени пользователя базы данных
### однако у нас в ОС нет пользователя testread, поэтому происходит отказ
### я поменяю в файле pg_hba.conf метод peer на trust и тогда можно будет подключиться к базе
### а select отработал, из-за того, что мы остались под предыдущим пользователем

```sql
postgres=# select setting from pg_settings where setting like '%pg_hba%';
               setting
-------------------------------------
 /etc/postgresql/16/main/pg_hba.conf
(1 row)

vim /etc/postgresql/16/main/pg_hba.conf
# "local" is for Unix domain socket connections only
local   all             all                                     trust

root@compute-vm-2-2-20-hdd-1734505245653:~# pg_ctlcluster
Error: Usage: /usr/bin/pg_ctlcluster <version> <cluster> <action> [-- <pg_ctl options>]
root@compute-vm-2-2-20-hdd-1734505245653:~# pg_ctlcluster 16 main restart
psql
postgres=# \c testdb testread
You are now connected to database "testdb" as user "testread".
testdb=>
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=>

```



