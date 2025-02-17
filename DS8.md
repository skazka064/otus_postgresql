#### Настройте выполнение контрольной точки раз в 30 секунд
```sql
postgres=# alter system set checkpoint_timeout ='30s';
ALTER SYSTEM

postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```
### 10 минут c помощью утилиты pgbench подавайте нагрузку

#### Инициализируем таблицы
```sql
postgres@Ubuntu:~$ pgbench -i postgres        
dropping old tables...
creating tables...
generating data...
100000 of 100000 tuples (100%) done (elapsed 0.10 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done.
```
#### Сбросим статистику
```sql
postgres=# select pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared 
----------------------
 
(1 row)
```
#### Узнаем предварительно позицию в журнале
```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 E/6A358628
(1 row)
```
#### Запускаем pgbench
```sql
postgres@Ubuntu:~$ pgbench -U postgres -T 600 
starting vacuum...end.

```
#### Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
```sql

```
