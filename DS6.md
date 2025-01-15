
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
