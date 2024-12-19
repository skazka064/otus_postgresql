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
### сделайте select * from t1;
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
### однако у нас в ОС нет пользователя testread, поэтому произошел отказ, а select отработал, из-за того, что мы остались под предыдущим пользователем
### я поменяю в файле pg_hba.conf метод peer на trust или md5 (т.к. мы задали пароль то можно и его), и тогда можно будет подключиться к базе

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
### select не отработал
```sql
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

```
###  оказалось, что таблица t1 создана в схеме public, а мы дали роли право на чтение таблиц только в схеме testnm
### попробуем дать, право для роли readonly использовать public, и посмотрим что получится
### посмотрим, также привилегии базы testdb
### для привилегии readonly есть привилегия connect(login), выдана postgres: readonly=c/postgres

```sql
testdb=> select * from pg_database where datname like '%test%' \gx
-[ RECORD 1 ]--+---------------------------------------------------------
oid            | 16388
datname        | testdb
datdba         | 10
encoding       | 6
datlocprovider | c
datistemplate  | f
datallowconn   | t
datconnlimit   | -1
datfrozenxid   | 722
datminmxid     | 1
dattablespace  | 1663
datcollate     | en_US.UTF-8
datctype       | en_US.UTF-8
daticulocale   |
daticurules    |
datcollversion | 2.39
datacl         | {=Tc/postgres,postgres=CTc/postgres,readonly=c/postgres}

postgres=# grant usage ON  schema public TO readonly ;
GRANT

postgres=# \c testdb testread
You are now connected to database "testdb" as user "testread".
testdb=>
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
### и... нет та же ошибка

### попробуем пойти другим путем, подключимся к базе testdb, где таблица t1 под пользователем postgres
### можно заметить, что при подключении к testdb под пользователем postgres доступ к таблице есть
```sql
postgres=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=> \c testdb postgres
You are now connected to database "testdb" as user "postgres".
testdb=# select * from t1;
 c1
----
  1
(1 row)
```
### значит посмотрим привилегии на таблицу t1
### и здесь мы видим, что владелец у t1 postgres, т.к. при создании таблицы мы были под пользователем postgres (You are now connected to database "testdb" as user "postgres".)

```sql
testdb=# select * from information_schema.table_privileges where table_name='t1' ;
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | postgres | testdb        | public       | t1         | INSERT         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | SELECT         | YES          | YES
 postgres | postgres | testdb        | public       | t1         | UPDATE         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | DELETE         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | TRUNCATE       | YES          | NO
 postgres | postgres | testdb        | public       | t1         | REFERENCES     | YES          | NO
 postgres | postgres | testdb        | public       | t1         | TRIGGER        | YES          | NO
(7 rows)
```

### попробуем поменять владельца у таблицы t1 на readonly
```sql
 \c testdb postgres
testdb=# alter table t1 owner to readonly ;
ALTER TABLE
testdb=# select * from information_schema.table_privileges where table_name='t1' ;
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 readonly | readonly | testdb        | public       | t1         | INSERT         | YES          | NO
 readonly | readonly | testdb        | public       | t1         | SELECT         | YES          | YES
 readonly | readonly | testdb        | public       | t1         | UPDATE         | YES          | NO
 readonly | readonly | testdb        | public       | t1         | DELETE         | YES          | NO
 readonly | readonly | testdb        | public       | t1         | TRUNCATE       | YES          | NO
 readonly | readonly | testdb        | public       | t1         | REFERENCES     | YES          | NO
 readonly | readonly | testdb        | public       | t1         | TRIGGER        | YES          | NO
(7 rows)


testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from t1;
 c1
----
  1
(1 row)

testdb=>

```
### теперь получилось работать с привилегией select для пользователя testread
### вернем владельца на postgres
```sql
testdb=# alter table t1 owner to postgres ;
ALTER TABLE
testdb=# select * from information_schema.table_privileges where table_name='t1' ;
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | postgres | testdb        | public       | t1         | INSERT         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | SELECT         | YES          | YES
 postgres | postgres | testdb        | public       | t1         | UPDATE         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | DELETE         | YES          | NO
 postgres | postgres | testdb        | public       | t1         | TRUNCATE       | YES          | NO
 postgres | postgres | testdb        | public       | t1         | REFERENCES     | YES          | NO
 postgres | postgres | testdb        | public       | t1         | TRIGGER        | YES          | NO
(7 rows)
```
### а теперь дадим роли readonly право select на все таблицы в схеме public
### и подключимся к testdb пользователем testread
### доступ к таблице получен
```sql
testdb=# grant select on all tables in schema public to readonly ;
GRANT
testdb=# select * from t1;
 c1
----
  1
(1 row)

testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from t1;
 c1
----
  1
(1 row)

```
### вернитесь в базу данных testdb под пользователем postgres
```sql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
### удалите таблицу t1
```sql
testdb=# drop table t1 ;
DROP TABLE
testdb=#
```
### создайте ее заново но уже с явным указанием имени схемы testnm
```sql
testdb=# create table testnm.t1 (c1 integer) ;
CREATE TABLE
```
### вставьте строку со значением c1=1
```sql
testdb=# insert into testnm.t1 values ('1');
INSERT 0 1
```
### зайдите под пользователем testread в базу данных testdb
```sql
testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=>
```
### сделайте select * from testnm.t1;
```sql
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
testdb=> \dt
Did not find any relations.
testdb=> show search_path
testdb-> ;
   search_path
-----------------
 "$user", public
(1 row)
```
### select не удался, посмотрим что в search_path
### добавим схему testnm в search_path
```sql
testdb=> set search_path to "$user",public,testnm
testdb-> ;
SET
testdb=> show search_path;
       search_path
-------------------------
 "$user", public, testnm
(1 row)
```
### посмотрим информацию о таблице 
```sql
testdb=> \dt+
                                     List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+------+-------+----------+-------------+---------------+------------+-------------
 testnm | t1   | table | postgres | permanent   | heap          | 8192 bytes |
(1 row)

```
### выполним  select * from information_schema.table_privileges where table_name='t1' ;
```sql
postgres=> \c postgres postgres
You are now connected to database "postgres" as user "postgres".
postgres=# select * from information_schema.table_privileges where table_name='t1' ;
 grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
(0 rows)

```
### запрос возвратил ноль строк, значит привелегии отсутствуют
### попробуем дать привилегии еще раз
```sql
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```
### на этот раз запрос отработал
### а до этого не отрабатывал, т.к. гранты действовали только на текущие объекты, а мы создали новую таблицу, уже после выдачи привелегий.

```sql
testdb=> select * from information_schema.table_privileges where table_name='t1' ;
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
(1 row)

testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)


```
### как сделать так чтобы такое больше не повторялось?
```sql
testdb=> alter default privileges in schema testnm grant select on tables to readonly ;
ALTER DEFAULT PRIVILEGES
```
### создадим новую таблицу, что бы это проверить
```sql
testdb=> \c testdb postgres
You are now connected to database "testdb" as user "postgres".
testdb=# create table testnm.t2 (c1 integer) ;
CREATE TABLE
testdb=# insert into testnm.t2 values (2);
INSERT 0 1
testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t2;
ERROR:  permission denied for table t2
```
### не получилось, потому что я создал таблицу t2 под пользователем postgres
### поменяем владельца 
```sql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# \dn
      List of schemas
  Name  |       Owner
--------+-------------------
 public | pg_database_owner
 testnm | postgres
(2 rows)

testdb=# \c postgres postgres
You are now connected to database "postgres" as user "postgres".
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# alter schema
information_schema  pg_catalog          pg_toast            public              testnm
testdb=# alter schema testnm
OWNER TO   RENAME TO
testdb=# alter schema testnm owner to testread ;
ALTER SCHEMA
testdb=# \dn
      List of schemas
  Name  |       Owner
--------+-------------------
 public | pg_database_owner
 testnm | testread
(2 rows)

testdb=# select * from testnm.t2;
 c1
----
  2
(1 row)

testdb=#
testdb=# select * from testnm.t1;
 c1
----
  1
(1 row)

```
### сейчас удалось выбрать данные из вновь созданной таблицы t2 c такой настройкой  alter default privileges in schema testnm grant select on tables to readonly , а так же и из t1
### теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```sql
testdb=# drop table testnm.t2;
DROP TABLE
testdb=# create table t2(c1 integer);
CREATE TABLE
testdb=# insert into t2 values (2);
INSERT 0 1
testdb=# sea

testdb=# show search_path ;
   search_path
-----------------
 "$user", public
(1 row)

testdb=#
```
### дело в том, что по умолчанию все объекты, если не указать схему создаются в схеме public, а на нее у нас и у всех есть разрешения.

### теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
```sql
testdb=# REVOKE CREATE on SCHEMA public FROM public;
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE

testdb=# \c testdb testread;
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=>  create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
testdb=> insert into t2 values (3);
ERROR:  permission denied for table t2
testdb=>
```
### а вот сейчас не получилось это сделать, потому что мы отобрали у public создавать в public а также отобрали все в базе testdb для public


