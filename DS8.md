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
#### Сбросим статистику, и будем записывать информацию о контрольных точках в лог
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
postgres=# alter system set log_checkpoints = on;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

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
#### Итак объем журнальных файлов 354 MB; У нас получилось 21 контрольная точка. В среднем на одну КТ приходился объем равный 354 / 22 = 16MB

<table>
 <thead><tr><th>Параметр</th><th>Значение</th><th>Комментарий</th></tr></thead>
 <tbody>
<tr>
 <td>checkpoints_timed</td><td>21</td><td>21 -запланировано 21 чекпоинт. Т.к. мы сбросили первоначальную статистику. Всего журнал нагенерировал 354MB объема данных. Журнал занимает 16MB, следовательно 354/16 примерно 21,22  контрольных точек.</td>
</tr>
  <tr>
 <td>checkpoints_req</td><td>0</td><td>Это контрольные точки по требованию(max_wal_size)</td>
</tr>
   <tr>
 <td>checkpoint_write_time</td><td>538831</td><td>Общее время, которое було затрачено на этап обработки контрольной точки, в котором файлы записываются на диск, в миллисекундах 539/60 равно 9 секунд. Одна КТ обрабатывалась в среднем 9 секунд</td>
</tr>
   <tr>
 <td>buffers_checkpoint</td><td>35392</td><td>страницы, сброшенные при контрольных точках</td>
</tr>
     <tr>
 <td>buffers_backend</td><td>1570</td><td>страницы, сброшенные обслуживающими процессами</td>
</tr>
    <tr>
 <td>buffers_clean</td><td>0</td><td>страницы, сброшенные процессом фоновой записи</td>
</tr>
 </tbody>
</table>

#### В хорошо настроенной системе значение buffers_backend должно быть существенно меньше, чем сумма buffers_checkpoint и buffers_clean.

#### Большое значение checkpoint_req (по сравнению с checkpoints_timed) говорит о том, что контрольные точки происходят чаще, чем предполагалось.
### Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```bash
2025-02-17 19:29:37.061 MSK [1254] LOG:  checkpoint starting: time
2025-02-17 19:30:04.077 MSK [1254] LOG:  checkpoint complete: wrote 1943 buffers (1.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.970 s, sync=0.012 s, total=27.017 s; sync files=5, longest=0.004 s, average=0.003 s; distance=20209 kB, estimate=23808 kB
2025-02-17 19:30:07.080 MSK [1254] LOG:  checkpoint starting: time
2025-02-17 19:30:34.079 MSK [1254] LOG:  checkpoint complete: wrote 2016 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.949 s, sync=0.019 s, total=27.000 s; sync files=11, longest=0.007 s, average=0.002 s; distance=19715 kB, estimate=23399 kB
2025-02-17 19:30:37.083 MSK [1254] LOG:  checkpoint starting: time
2025-02-17 19:31:04.082 MSK [1254] LOG:  checkpoint complete: wrote 1913 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.955 s, sync=0.012 s, total=26.999 s; sync files=5, longest=0.007 s, average=0.003 s; distance=20401 kB, estimate=23099 kB
2025-02-17 19:31:07.082 MSK [1254] LOG:  checkpoint starting: time
2025-02-17 19:31:34.043 MSK [1254] LOG:  checkpoint complete: wrote 2035 buffers (1.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.933 s, sync=0.012 s, total=26.961 s; sync files=11, longest=0.006 s, average=0.002 s; distance=20117 kB, estimate=22801 kB
2025-02-17 19:31:37.047 MSK [1254] LOG:  checkpoint starting: time
2025-02-17 19:32:04.047 MSK [1254] LOG:  checkpoint complete: wrote 1911 buffers (1.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.943 s, sync=0.015 s, total=27.001 s; sync files=5, longest=0.008 s, average=0.003 s; distance=21011 kB, estimate=22622 kB
2025-02-17 19:32:07.050 MSK [1254] LOG:  checkpoint starting: time
2025-02-17 19:32:34.117 MSK [1254] LOG:  checkpoint complete: wrote 2053 buffers (1.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=27.017 s, sync=0.022 s, total=27.068 s; sync files=14, longest=0.008 s, average=0.002 s; distance=20456 kB, estimate=22405 kB
```
####  Контрольные точки выполнялись чаще, чем раз в 30 секунд. Потому что шла интенсивная работа с БД и журналы в 16MB заполнялись быстрее, и КТ происходили по мере заполнения журналов.  

### Сравните tps в синхронном/асинхронном режиме утилитой pgbench. 
```sql
postgres=# show synchronous_commit;      
 synchronous_commit 
--------------------
 on
(1 row)
```
#### Выключим synchronous_commit
```sql
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres@Ubuntu:~$ pgbench -U postgres -T 30 
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 30 s
number of transactions actually processed: 80232
latency average = 0.374 ms
tps = 2674.375552 (including connections establishing)
tps = 2674.553018 (excluding connections establishing)
```

### Объясните полученный результат
#### synchronous_commit определяет, после завершения какого уровня обработки WAL сервер будет сообщать об успешном выполнении операции. Допустимые значения remote_apply(применено удаленно), on(включео, по умолчанию), remote_write(записано удаленно), local(локально), off(выключено). Локальное действие всех отличных от off режимов заключается в ожидании локального сброса WAL на диск. В режиме off ожидание отсутствует, и поэтому образуется окно от момента, когда клиент узнает об успешном завершении, до момента, когда транзакция действительно зафиксирует свои результаты на диск. Максимальный размер этого окна равен wal_writer_delay * 3 Однако значение off не угрожает целостности данных. Сбой ОС или БД может привести к потере последних транзакций, считавшихся зафиксированными, но состояние БД будет точно таким же, как и в случае штатного прерывания этих транзакций. Следовательно увеличение TPS произошло за счет отсутствия времени ожидания подтверждения сброса WAL на диск.

### Создайте новый кластер с включенной контрольной суммой страниц
``` bash
postgres@Ubuntu:~$ /usr/lib/postgresql/12/bin/pg_controldata -D /var/lib/postgresql/12/main/ | grep checksum
Data page checksum version:           1
```
```sql
postgres=# show data_checksums ;
 data_checksums 
----------------
 on
(1 row)

```
###  Создайте таблицу. Вставьте несколько значений.
```sql
postgres=# create table t (s text);
CREATE TABLE
postgres=# insert into t values ('qw'),('er'),('ty');
INSERT 0 3

```
### Выключите кластер
```bash
root@Ubuntu:~# pg_ctlcluster 12 main stop
```
###  Измените пару байт в таблице
#### Найдем файл, где лежит таблица
```bash
postgres=# select pg_relation_filepath('t');
 pg_relation_filepath 
----------------------
 base/13465/238836
(1 row)
```
#### Остановим кластер

```bash
root@Ubuntu:~# pg_ctlcluster 12 main stop
```
#### Поменяем несколько байт в странице(сотрем из заголовка LSN последней журнальной записи)
```bash
postgres@Ubuntu:~$ dd if=/dev/zero of=/var/lib/postgresql/12/main/base/13465/238836 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0,00757159 s, 1,1 kB/s

```
### Включите кластер и сделайте выборку из таблицы.
```bash
root@Ubuntu:~# pg_ctlcluster 12 main start
postgres=# select * from t;
WARNING:  page verification failed, calculated checksum 35578 but expected 11750
ERROR:  invalid page in block 0 of relation base/13465/238836
```
### Что и почему произошло?
#### Так как страница уже записалась на диск и была вытеснена из кэша произошло ее повреждение 
#### По умолчанию страницы данных не защищены контрольными суммами, но при необходимости можно их включить. В таком случае каждая страница данных будет содержать контрольную сумму, рассчитываемую при записи и проверяемую при каждом чтении страницы(что приведет к увеличению накладных расходов). Соответственно мы изменили внутреннюю структуру файла, и при чтении таблицы посчиталась контрольная сумма файла, которая не совпала с заявленной. 
### как проигнорировать ошибку и продолжить работу?
#### При попытке восстановить страницы после повреждения иногда нужно обойти защиту, обеспечиваемую контрольными суммами. Для этого можно временно установить параметр конфигурации ignore_checksum_failure. Параметр ignore_checksum_falure позволяет попробовать прочитать таблицу с риском получить искаженные данные. Это можно сделать, если , например у нас нет резервной копии.
```sql
postgres=# set ignore_checksum_failure =on;
SET
postgres=# select * from t;
WARNING:  page verification failed, calculated checksum 35578 but expected 11750
 s  
----
 qw
 er
 ty
(3 rows)

```
