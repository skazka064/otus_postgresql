# Начало
#### развернуть виртуальную машину любым удобным способом
#### поставить на неё PostgreSQL 15 любым способом
#### настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
## Настройки параметров по умолчанию такие:
```sql
postgres=# show work_mem;
 work_mem
----------
 4MB
(1 row)
```
#### используется для сортировок, построения хэш таблиц. Это позволяет выполнять данные операции в памяти, что гораздо быстрее обращений к диску. В рамках одного запроса данный параметр может быть использован много раз. Если ваш запрос содержит пять операций сортировки, то память которая потребуется для его выполнения как минимум 5*work_mem.  Так как, скорее всего на сервере вы не одни и сессий много, то каждая из них может использовать этот параметр по многу  раз, поэтому не рекомендуется делать его слишком большим. Можно выставить небольшое значение в глобальном конфиге, а в сессиях локально его менять(в случае сложных запросов).
```sql
postgres=# show effective_cache_size ;
 effective_cache_size
----------------------
 4GB
(1 row)
```
#### служит подсказкой для планировщика, сколько ОП у него в запасе. Можно определить как shared_buffers + ОП системы, т.е. ОП, используемое самой ОС и сторонними приложениями. За счет данного параметра планировщик чаще может использовать индексы. Наиболее часто используемое значение 75% от общей на сервере.
```sql
postgres=# show shared_buffers ;
 shared_buffers
----------------
 128MB
(1 row)
```
#### используется для кэширования данных. По умолчанию значение низкое- для поддержания большого количества ОС. Начать стоит с его изменения. Согласно документации, рекомендуемое значение для этого параметра 25% от общей оперативной памяти на сервере. Postgresql использует два кэша. Свой shared_buffers и ОС. Редко значение больше чем 40% окажет влияние на производительность.
```sql
postgres=# show max_connections ;
 max_connections
-----------------
 100
(1 row)
```
#### максимальное количество соединений. Для изменения параметра придется перезапускать сервер. Если пранируется использовать Postgresql как DWH то большого количества соединений не нужно. Данный параметр тесно связан с work_mem
```sql
postgres=# show maintenance_work_mem ;
 maintenance_work_mem
----------------------
 64MB
(1 row)
```
#### определяет максимальное количество ОП для операций VACUUM, CREATE INDEX, CREATE FOREGN KEY увеличение этого параметра позволит быстее выполнять эти операции. Не связано с work_mem , поэтому можно ставить в разы больше, чем work_mem
```sql
postgres=# show wal_buffers ;
 wal_buffers
-------------
 4MB
(1 row)
```
#### объем разделяемой памяти, которая будет использоваться для буфферизации данных wal, еще не записанных на диск. Если у вас большое количество одновременных подключений, то увеличение параметра улучшит производительность. По умолчанию -1 определяется автоматически, как 1/32 от shared_buffers, но не больше чем 16МБ. В ручную можно задавать большие значения. Но обычно не боьше 16МБ
```sql
postgres=# show checkpoint_timeout ;
 checkpoint_timeout
--------------------
 5min
(1 row)
```
#### чем реже происходит сбрасывание, тем дольше будет восстановление БД после сбоя. Значение по умолчанию 5 минут, рекомендуемое от 30 минут до часа.
```sql
postgres=# show max_wal_size ;
 max_wal_size
--------------
 1GB
(1 row)
```
#### максимальный размер  до которого может возрастать wal между контрольными точками в wal. Значение по умолчанию 1ГБ. Увеличение этого параметра может привести к увеличению времени, которое понадобится для восстановления после сбоя. Но позволит реже выполнять операцию сбрасывания на диск. Так же сбрасывание может выполнится и при достижении нужного времени, определенного в параметре checkpoint_timeout
#### Найдем расположение конфиг файла через psql 
```sql
postgres=# show config_file ;
               config_file
-----------------------------------------
 /etc/postgresql/12/main/postgresql.conf

postgres=# select current_setting('config_file');
             current_setting
-----------------------------------------
 /etc/postgresql/12/main/postgresql.conf
(1 row)

```
#### запустим pgbench сначала с дефолтным конфигом
```sql
postgres@Ubuntu:~$ pgbench -c 50 -j 2 -P 10 -T 60
starting vacuum...end.
progress: 10.0 s, 1016.1 tps, lat 48.672 ms stddev 50.000
progress: 20.0 s, 722.7 tps, lat 69.077 ms stddev 78.785
progress: 30.0 s, 783.3 tps, lat 63.720 ms stddev 84.850
progress: 40.0 s, 987.6 tps, lat 50.691 ms stddev 53.640
progress: 50.0 s, 1052.8 tps, lat 47.532 ms stddev 48.866
progress: 60.0 s, 1067.2 tps, lat 46.823 ms stddev 50.160
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 56347
latency average = 53.215 ms
latency stddev = 61.207 ms
tps = 937.638183 (including connections establishing)
tps = 937.678190 (excluding connections establishing)
```
#### теперь попробуем настроить сервер на максимальную производительность. Будем это делать через pgconfig
### General WEB Aplication

<table>
 <th>Num</th>
 <th>Значение</th>
  <th>NEW</th>
 <th>DEFAULT</th>
  <tr>
   <td>1</td>
   <td>shared_buffers</td>
   <td>1GB</td>
   <td>128MB</td>
  </tr>
   <tr>
     <td>2</td>
   <td>work_mem</td>
   <td>10MB</td>
   <td>4MB</td>
  </tr>
 <tr>
   <td>3</td>
   <td>effective_cache_size</td>
   <td>3GB</td>
   <td>4GB</td>
  </tr> 
   <tr>
     <td>4</td>
   <td>max_connections</td>
   <td>100</td>
   <td>100</td>
  </tr> 
 <tr>
  <td>5</td>
   <td>max_wal_size</td>
   <td>3GB</td>
   <td>1GB</td>
  </tr> 
   <tr>
  <td>6</td>
   <td>checkpoint_timeout</td>
   <td>5min</td>
   <td>5min</td>
  </tr> 
   <tr>
  <td>7</td>
   <td>wal_buffers</td>
   <td>-1</td>
   <td>4MB</td>
  </tr> 
</table>

```sql
postgres@Ubuntu:~$ pgbench -c 50 -j 2 -P 10 -T 60
starting vacuum...end.
progress: 10.0 s, 1013.0 tps, lat 48.878 ms stddev 54.439
progress: 20.0 s, 1047.4 tps, lat 47.703 ms stddev 50.862
progress: 30.0 s, 1039.7 tps, lat 47.945 ms stddev 51.288
progress: 40.0 s, 1073.1 tps, lat 46.700 ms stddev 51.368
progress: 50.0 s, 1074.7 tps, lat 46.583 ms stddev 52.183
progress: 60.0 s, 1063.2 tps, lat 46.855 ms stddev 51.130
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 63161
latency average = 47.484 ms
latency stddev = 51.954 ms
tps = 1050.774504 (including connections establishing)
tps = 1050.811858 (excluding connections establishing)
```
#### количество tps увеличилось на 10% от дефолтных
#### теперь попробуем настройки OLTP
<table>
 <th>Num</th>
 <th>Значение</th>
  <th>NEW</th>
 <th>DEFAULT</th>
  <tr>
   <td>1</td>
   <td>shared_buffers</td>
   <td>1GB</td>
   <td>128MB</td>
  </tr>
   <tr>
     <td>2</td>
   <td>work_mem</td>
   <td>14MB</td>
   <td>4MB</td>
  </tr>
 <tr>
   <td>3</td>
   <td>effective_cache_size</td>
   <td>3GB</td>
   <td>4GB</td>
  </tr> 
   <tr>
     <td>4</td>
   <td>max_connections</td>
   <td>100</td>
   <td>100</td>
  </tr> 
 <tr>
  <td>5</td>
   <td>max_wal_size</td>
   <td>3GB</td>
   <td>1GB</td>
  </tr> 
   <tr>
  <td>6</td>
   <td>checkpoint_timeout</td>
   <td>5min</td>
   <td>5min</td>
  </tr> 
   <tr>
  <td>7</td>
   <td>wal_buffers</td>
   <td>-1</td>
   <td>4MB</td>
  </tr> 
</table>

```sql
postgres@Ubuntu:~$  pgbench -c 50 -j 2 -P 10 -T 60
starting vacuum...end.
progress: 10.0 s, 1059.6 tps, lat 46.626 ms stddev 50.903
progress: 20.0 s, 691.3 tps, lat 72.126 ms stddev 81.309
progress: 30.0 s, 863.6 tps, lat 58.004 ms stddev 66.803
progress: 40.0 s, 805.3 tps, lat 62.058 ms stddev 70.010
progress: 50.0 s, 933.1 tps, lat 53.747 ms stddev 60.760
progress: 60.0 s, 957.1 tps, lat 52.101 ms stddev 59.287
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 53150
latency average = 56.443 ms
latency stddev = 64.832 ms
tps = 884.029625 (including connections establishing)
tps = 884.058544 (excluding connections establishing)

```
#### количество tps уменьшилось на 6% от дефолтных и 16% от предыдущих (WEB)
#### 
