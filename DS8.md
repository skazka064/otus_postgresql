### Настройте выполнение контрольной точки раз в 30 секунд
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
postgres=# SELECT * FROM pg_stat_bgwriter\gx
-[ RECORD 1 ]---------+-----------------------------
checkpoints_timed     | 0
checkpoints_req       | 0
checkpoint_write_time | 0
checkpoint_sync_time  | 0
buffers_checkpoint    | 0
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 0
buffers_backend_fsync | 0
buffers_alloc         | 0
stats_reset           | 2025-02-17 17:56:19.17317+03

```
#### Узнаем предварительно позицию в журнале
```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 E/9BD32AE0
(1 row)

```
#### Запускаем pgbench
```sql
postgres@Ubuntu:~$ pgbench -U postgres -T 600 
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 244879
latency average = 2.450 ms
tps = 408.129466 (including connections establishing)
tps = 408.131814 (excluding connections establishing)

```
### Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
#### Для этого найдем текущий lsn и вычтем его из найденного ранее lsn
```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 E/B1ED2760
(1 row)

postgres=# select pg_size_pretty('E/B1ED2760'::pg_lsn-'E/9BD32AE0'::pg_lsn);    
 pg_size_pretty 
----------------
 354 MB
(1 row)

postgres=# SELECT * FROM pg_stat_bgwriter\gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 21
checkpoints_req       | 0
checkpoint_write_time | 538831
checkpoint_sync_time  | 328
buffers_checkpoint    | 35392
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 1570
buffers_backend_fsync | 0
buffers_alloc         | 1563
stats_reset           | 2025-02-17 18:06:50.890768+03

```
#### Итак 
<table>
 <thead><tr><th>Параметр</th><th>Значение</th><th>Комментарий</th></tr></thead>
 <tbody>
<tr>
 <td>checkpoints_timed</td><td>21</td><td>21 -запланировано 21 чекпоинт. Всего журнал нагенерировал 354MB объема данных. Журнал занимает 16MB, следовательно 354/16 примерно 21,22  контрольных точек.</td>
</tr>
  <tr>
 <td>checkpoints_req</td><td>0</td><td>Это количество запрошенных контрольных точек, которые были уже выполнены, ноль т.к. до теста мы сбросили статистику.</td>
</tr>
 </tbody>

</table>

