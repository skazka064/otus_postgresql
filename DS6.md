
### Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.
#### Проверим, включено ли у нас логирование. 
#### Логирование включено
```sql
postgres=# show log_lock_waits ;
 log_lock_waits
----------------
 on
(1 row)

```
#### Посмотрим log_min_duration_statement. Управляет тем, какие SQL-операторы, завершившиеся ошибкой, записываются в журнал сервера. SQL-оператор будет записан в журнал, если он завершится ошибкой с указанным уровнем важности или выше. Допустимые значения: DEBUG5, DEBUG4, DEBUG3, DEBUG2, DEBUG1, INFO, NOTICE, WARNING, ERROR, LOG, FATAL и PANIC. По умолчанию используется ERROR. Это означает, что в журнал сервера будут записаны все операторы, завершившиеся сообщением с уровнем важности ERROR, LOG, FATAL и PANIC. Чтобы фактически отключить запись операторов с ошибками, установите для этого параметра значение PANIC. Изменить этот параметр могут только суперпользователи.
```sql
postgres=# show log_min_duration_statement ;
 log_min_duration_statement
----------------------------
 10s
(1 row)
```

#### Проверим, через какое время срабатывает освобождение от взаимоблокировок
#### Через 1 сек
```sql
locks=# show deadlock_timeout ;
 deadlock_timeout
------------------
 1s
(1 row)
```
```sql
postgres=# alter system set deadlock_timeout ='200ms';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=#

```
#### воспроизведем блокировку
#### транзакция 1
```sql
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# \dt
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | accounts | table | postgres
(1 row)

locks=# begin;
BEGIN
locks=# UPDATE accounts SET amount = 10.00 WHERE acc_no = 1;
UPDATE 1
locks=# SELECT pg_sleep(1);
 pg_sleep
----------

(1 row)

locks=#  COMMIT;
COMMIT
locks=#
```
#### транзакция 2
```sql
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
locks=# begin;
BEGIN
locks=#  UPDATE accounts SET amount = 100.00 WHERE acc_no = 1;
UPDATE 1
locks=# COMMIT;
COMMIT
locks=#
```
#### посмотрим логи
```bash
postgres@Ubuntu:~$ tail -n 7  /var/lib/postgresql/12/main/log/postgresql-2025-01-15_114953.csv
2025-01-15 12:55:42.219 MSK,"postgres","locks",18264,"[local]",6787861e.4758,2,"authentication",2025-01-15 12:55:42 MSK,5/213,0,LOG,00000,"connection authorized: user=postgres database=locks application_name=psql",,,,,,,,,""
2025-01-15 12:55:42.221 MSK,"postgres","postgres",18260,"[local]",67878608.4754,3,"idle",2025-01-15 12:55:20 MSK,,0,LOG,00000,"disconnection: session time: 0:00:22.052 user=postgres database=postgres host=[local]",,,,,,,,,"psql"
2025-01-15 12:56:37.503 MSK,"postgres","locks",18264,"[local]",6787861e.4758,3,"UPDATE waiting",2025-01-15 12:55:42 MSK,5/214,6207431,LOG,00000,"process 18264 still waiting for ShareLock on transaction 6207430 after 203.945 ms","Process holding the lock: 18262. Wait queue: 18264.",,,,"while updating tuple (0,7) in relation ""accounts""","UPDATE accounts SET amount = 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 12:56:55.507 MSK,"postgres","locks",18264,"[local]",6787861e.4758,4,"UPDATE waiting",2025-01-15 12:55:42 MSK,5/214,6207431,LOG,00000,"process 18264 acquired ShareLock on transaction 6207430 after 18207.615 ms",,,,,"while updating tuple (0,7) in relation ""accounts""","UPDATE accounts SET amount = 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 12:56:55.507 MSK,"postgres","locks",18264,"[local]",6787861e.4758,5,"UPDATE",2025-01-15 12:55:42 MSK,5/214,6207431,LOG,00000,"duration: 18208.860 ms  statement: UPDATE accounts SET amount = 100.00 WHERE acc_no = 1;",,,,,,,,,"psql"
2025-01-15 12:59:58.632 MSK,,,1305,,678776b3.519,3,,2025-01-15 11:49:55 MSK,,0,LOG,00000,"checkpoint starting: time",,,,,,,,,""
2025-01-15 12:59:58.760 MSK,,,1305,,678776b3.519,4,,2025-01-15 11:49:55 MSK,,0,LOG,00000,"checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.105 s, sync=0.004 s, total=0.129 s; sync files=1, longest=0.004 s, average=0.004 s; distance=1 kB, estimate=1 kB",,,,,,,,,""

```
![Иллюстрация к проекту](img/2025-01-15_13-06-56.png)

### Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 
#### для этого создадим таблицу 
#### заполним ее данными
```sql
locks=# create table test_no_pk(acc_no integer, amount numeric);
CREATE TABLE
locks=# INSERT INTO test_no_pk VALUES (1,1000.00),(2,2000.00),(3,3000.00);
INSERT 0 3
```
#### для удобства создадим представление
```sql
locks=# CREATE VIEW locks AS
locks-# SELECT pid,
locks-#        locktype,
locks-#        CASE locktype
locks-#          WHEN 'relation' THEN relation::REGCLASS::text
locks-#          WHEN 'virtualxid' THEN virtualxid::text
locks-#          WHEN 'transactionid' THEN transactionid::text
locks-#          WHEN 'tuple' THEN relation::REGCLASS::text||':'||tuple::text
locks-#        END AS lockid,
locks-#        mode,
locks-#        granted
locks-# FROM pg_locks;
CREATE VIEW
```
#### итак, первая транзакция обновляет, и соответственно блокирует строку
```sql
locks=# begin;
BEGIN
locks=# select txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      6207436 |          18262
(1 row)

locks=# UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1

```
#### вторая транзакция делает то же самое
```sql
locks=# begin;
BEGIN
locks=# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      6207437 |          18264
(1 row)

locks=#  UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;
```
#### И третья
```sql
locks=# begin;
BEGIN
locks=# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
      6207438 |          19055
(1 row)

locks=# UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;
```
####  Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
#### Блокировки для первой транзакции:
```sql
locks=# select * from locks where pid=18262;
  pid  |   locktype    |              lockid               |       mode       | granted
-------+---------------+-----------------------------------+------------------+---------
 18262 | relation      | pg_locks                          | AccessShareLock  | t
 18262 | relation      | locks                             | AccessShareLock  | t
 18262 | relation      | test_no_pk                        | RowExclusiveLock | t
 18262 | relation      | pg_class_tblspc_relfilenode_index | AccessShareLock  | t
 18262 | relation      | pg_class_relname_nsp_index        | AccessShareLock  | t
 18262 | relation      | pg_class_oid_index                | AccessShareLock  | t
 18262 | relation      | pg_namespace_oid_index            | AccessShareLock  | t
 18262 | relation      | pg_namespace_nspname_index        | AccessShareLock  | t
 18262 | relation      | pg_namespace                      | AccessShareLock  | t
 18262 | relation      | pg_class                          | AccessShareLock  | t
 18262 | relation      | accounts_acc_no_idx1              | RowExclusiveLock | t
 18262 | relation      | accounts_acc_no_idx               | RowExclusiveLock | t
 18262 | relation      | accounts_pkey                     | RowExclusiveLock | t
 18262 | relation      | accounts                          | RowExclusiveLock | t
 18262 | virtualxid    | 7/22                              | ExclusiveLock    | t
 18262 | transactionid | 6207436                           | ExclusiveLock    | t
(16 rows)
```
* Тип relation для pg_locks locks pg_class pg_namespace в режиме AccessShareLock - установились на читаемые отношения
* Тип relation для test_no_pk в режиме RowExclusiveLock - установились на изменяемую таблицу
* Типы virtualxid transactionid в режиме ExclusiveLock - удерживаются каждой транзакцией для самой себя
#### Блокировки для второй транзакции
```sql
locks=# select * from locks where pid=18264;
  pid  |   locktype    |    lockid    |       mode       | granted
-------+---------------+--------------+------------------+---------
 18264 | relation      | test_no_pk   | RowExclusiveLock | t
 18264 | virtualxid    | 5/216        | ExclusiveLock    | t
 18264 | tuple         | test_no_pk:1 | ExclusiveLock    | t
 18264 | transactionid | 6207437      | ExclusiveLock    | t
 18264 | transactionid | 6207436      | ShareLock        | f
(5 rows)

```
* Блокировки AccessShareLock отсутствуют, т.к. мы не обращались запросами на чтение
* Транзакция ожидает получене блокировки типа transactionid в режиме ShareLock(granted=f) txid первой транзакции=6207436 здесь видно, что вторая транзакция не получила блокировку номера первой транзакции, т.к. первая транзакция еще длится
* Появилась блокировка типа tuple для обновляемой строки
#### Блокировки для третей транзакции
```sql
locks=# select * from locks where pid=19055;
  pid  |   locktype    |              lockid               |       mode       | granted
-------+---------------+-----------------------------------+------------------+---------
 19055 | relation      | test_no_pk                        | RowExclusiveLock | t
 19055 | relation      | pg_class_tblspc_relfilenode_index | AccessShareLock  | t
 19055 | relation      | pg_class_relname_nsp_index        | AccessShareLock  | t
 19055 | relation      | pg_class_oid_index                | AccessShareLock  | t
 19055 | relation      | pg_namespace_oid_index            | AccessShareLock  | t
 19055 | relation      | pg_namespace_nspname_index        | AccessShareLock  | t
 19055 | relation      | pg_namespace                      | AccessShareLock  | t
 19055 | relation      | pg_class                          | AccessShareLock  | t
 19055 | virtualxid    | 8/4                               | ExclusiveLock    | t
 19055 | tuple         | test_no_pk:1                      | ExclusiveLock    | f
 19055 | transactionid | 6207438                           | ExclusiveLock    | t
(11 rows)

```
* А вот третья транзакция уже увидела, что вторая транзакция попыталась получить блокировку номера первой транзакции, и у нее не получилось, а так же она видит что вторая транзакция получила блокировку типа tuple этой же строки, и поэтому она тоже берет блокировку типа tuple этой самой строки
#### общая картина
```sql
locks=# SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
locks-# FROM pg_stat_activity
locks-# WHERE backend_type = 'client backend';
  pid  | wait_event_type |  wait_event   | pg_blocking_pids
-------+-----------------+---------------+------------------
  1568 | Client          | ClientRead    | {}
  1569 | Client          | ClientRead    | {}
 18264 | Lock            | transactionid | {18262}
 18262 |                 |               | {}
 19055 | Lock            | tuple         | {18264}
(5 rows)
```
* Здесь видно, что транзакция с pid 18264 заблокирована транзакцией 18262 тип transactionid
* А транзакция с pid 19055 заблокирована транзакцией с pid 18264 тип tuple
```bash
postgres@Ubuntu:~$ tail -n 7  /var/lib/postgresql/12/main/log/postgresql-2025-01-15_114953.csv
2025-01-15 13:37:23.551 MSK,"postgres","locks",19055,"[local]",67878fbb.4a6f,3,"UPDATE waiting",2025-01-15 13:36:43 MSK,8/4,6207438,LOG,00000,"process 19055 still waiting for ExclusiveLock on tuple (0,1) of relation 189621 of database 173237 after 200.835 ms","Process holding the lock: 18264. Wait queue: 19055.",,,,,"UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 13:45:51.840 MSK,"postgres","locks",18264,"[local]",6787861e.4758,8,"UPDATE waiting",2025-01-15 12:55:42 MSK,5/216,6207437,LOG,00000,"process 18264 acquired ShareLock on transaction 6207436 after 1253993.651 ms",,,,,"while updating tuple (0,1) in relation ""test_no_pk""","UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 13:45:51.840 MSK,"postgres","locks",19055,"[local]",67878fbb.4a6f,4,"UPDATE waiting",2025-01-15 13:36:43 MSK,8/4,6207438,LOG,00000,"process 19055 acquired ExclusiveLock on tuple (0,1) of relation 189621 of database 173237 after 508489.910 ms",,,,,,"UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 13:45:51.840 MSK,"postgres","locks",18264,"[local]",6787861e.4758,9,"UPDATE",2025-01-15 12:55:42 MSK,5/216,6207437,LOG,00000,"duration: 1253994.120 ms  statement: UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,,,,,,,"psql"
2025-01-15 13:45:52.041 MSK,"postgres","locks",19055,"[local]",67878fbb.4a6f,5,"UPDATE waiting",2025-01-15 13:36:43 MSK,8/4,6207438,LOG,00000,"process 19055 still waiting for ShareLock on transaction 6207437 after 201.164 ms","Process holding the lock: 18264. Wait queue: 19055.",,,,"while rechecking updated tuple (0,4) in relation ""test_no_pk""","UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 13:45:57.978 MSK,"postgres","locks",19055,"[local]",67878fbb.4a6f,6,"UPDATE waiting",2025-01-15 13:36:43 MSK,8/4,6207438,LOG,00000,"process 19055 acquired ShareLock on transaction 6207437 after 6137.409 ms",,,,,"while rechecking updated tuple (0,4) in relation ""test_no_pk""","UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,"psql"
2025-01-15 13:45:57.978 MSK,"postgres","locks",19055,"[local]",67878fbb.4a6f,7,"UPDATE",2025-01-15 13:36:43 MSK,8/4,6207438,LOG,00000,"duration: 514627.989 ms  statement: UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;",,,,,,,,,"psql"

```
![Иллюстрация к проекту](img/2025-01-15_13-46-38.png)

#### Воспроизведите взаимоблокировку трех транзакций. 
```sql
Транзакция 1
 begin;
 update test_no_pk set amount =amount-100 where acc_no=1;
Транзакция 2
begin;
 update test_no_pk set amount =amount-100 where acc_no=2;
Транзакция 3
begin;
 update test_no_pk set amount =amount-100 where acc_no=3;

Транзакция 1
update test_no_pk set amount =amount+100 where acc_no=2;
Транзакция 2
update test_no_pk set amount =amount+100 where acc_no=3;
Транзакция 3
 UPDATE test_no_pk SET amount = amount + 100.00 WHERE acc_no = 1;
ERROR:  deadlock detected
DETAIL:  Process 4508 waits for ShareLock on transaction 6207443; blocked by process 4496.
Process 4496 waits for ShareLock on transaction 6207444; blocked by process 4506.
Process 4506 waits for ShareLock on transaction 6207445; blocked by process 4508.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,7) in relation "test_no_pk"
```
####  Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?










