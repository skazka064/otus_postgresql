## psql https://www.interdb.jp/pg/
   - systemctl list-units --type=service - посмотреть какие сервисы загружены
   - journalctl -xeu postgresql@14-main.service - лог конкретного сервиса 
```sql
psql -h server -d bd -p port -U user
-w, --no-password
-W, --password
-c, --command - запускаем с одной sql командой
-f, --file - выполняем комманды из файла
-l, --list - смотрим доступные базы
-v, --set - 
-X, --no-psqlrc - не читаем startup file (~/.psqllrc)
-o, --output =FILENAME
\a, -A - вывод без выравнивания
\t, вывод без загоровков столбца
/usr/local/pgsql/etc/psqlrc, ~.psqlrc - при запуске psql выполняются комады записанные в этих файлах
\conninfo
select * from pg_database;
\l -список баз данных
\set ECHO_HIDDEN on - расскажет каким запросом он получил вывод команды psql
\x - переключаем режим вывода- столбцы транспонируем в строки
SHOW search_path;
 SELECT current_shema();
Схема
pg_catalog; public
```
# Настройка производительности Postgresql
### Уровни настройки
- Уровень сервера **(postgresql.conf postgresql.auto.conf)**
- Уровень пользователя/роли **(ALTER ROLE postgres SET work_mem TO '256MB')**
- Уровень Базы Данных **(ALTER DATABASE postgres SET work_mem TO '128MB')**
- Уровень сеанса **(set work_mem ='256MB')**
- Уровень процедуры/функции **(CREATE FUNCTION .... SET configuration_parameter TO value)**
### Посмотреть настройки сеанса
- show all
- explain (analyze, settings) select * from pgbench_accounts; **Посмотреть, какие настройки скручены.**

### Настраиваем сервер сограсно типу нагрузки ( https://pgtune.leopard.in.ua/ )
### Создаем тестовую базу ( pgbench -i -s 10 ) (pgbench -c 10 -j 1 -t 5000)
### 1. Центральный процессор
   - max_worker_processes = 28
   - max_parallel_workers_per_gather = 4
   - max_parallel_workers = 28
   - max_parallel_maintenance_workers = 4
   - top (смотрим параметр ID - сколько процессорных ресурсов находятся в резерве, должно быть не менее 20%)
### 2. Память
  ### ![Memory](/img/pg_memory.png)
 -  ps auxf | grep postgres; ps fp $(pgrep post) - можно посмотреть количество подключений к базе
 -  top - смотрим MiB Mem free свободной памяти должно быть не менее 10 % от общего объема Swap- использоваться не должен 0.0 used
 -  SELECT * FROM pg_stat_database (Смотрим строки blks_read (сколько данных было взято напрямую с диска) blks_hit (сколько данных было взято  из кэша должнл бять не менне 90% от общего количества обращений) )
        Если это значение меньше, то необходимо добавить shared_buffers
 - То же самое можно получить на уровне отдельной таблицы  SELECT * FROM pg_statio_user_tables (heap_blks_read heap_blks_hit)
 - EXPLAIN(ANALYSE, buffers) SELECT * FROM pgbench_accounts; (Buffers: shared hit=16643 read=79)
 - shared_preload_libraries = 'pg_prewarm' ; pg_prewarm.autoprewarm = true ;
 - EXPLAIN (ANALYSE) SELECT * FROM pgbench_accounts order by abalance;
    --     Sort Method: external merge  Disk: 39944kB
   --		  ->  Sort  (cost=86535.67..87578.21 rows=417016 width=97) (actual time=698.008..735.665 rows=333333 loops=3)
   --      Execution Time: 1214.646 ms
   SET work_mem='200MB'
   --   Sort Method: quicksort  Memory: 165202kB
   --  Sort  (cost=126477.78..128979.88 rows=1000838 width=97) (actual time=338.503..439.494 rows=1000000 loops=1)
### 3. Диск (iostat -xmt параметр aqu-sz - это очередь к диску, десятки и сотни это уже много)
   - effective_io_concurrency насколько можно распараллеливать обращение к диску

### 4. Сеть (tc -s -d qdisc ls dev enp0s3)
   - смотрим параметр backlog 0b 0p - это сколько байт  и пакетов стоит на отправку
 
 # Настройка Postgresql для работы с запросами 
### 1. Log_statement
   
 ```sql
-- Role: monitoring
-- DROP ROLE IF EXISTS monitoring;

CREATE ROLE monitoring WITH
  LOGIN
  SUPERUSER
  INHERIT
  CREATEDB
  CREATEROLE
  NOREPLICATION
  BYPASSRLS
  ENCRYPTED PASSWORD 'SCRAM-SHA-256$4096:h0IQkNOaLdHBZxCQ+6+r5Q==$d7XJxILOgrS3nOGNGEIOJ8/id/44R8CcEiZRMUfWyiU=:In2wXNKLt2QqQ1XxJmLoykiPck0BA87Y321nh8NnL1A=';

ALTER ROLE monitoring IN DATABASE postgres SET log_statement TO 'all';
cat /var/log/postgresql/postgresql-14-main.log

```
### 2. Auto_explain
   
 ```sql
ALTER ROLE monitoring IN DATABASE postgres
    SET "auto_explain.log_analyze" TO 'true';
ALTER ROLE monitoring IN DATABASE postgres
    SET "auto_explain.log_buffers" TO 'true';
ALTER ROLE monitoring IN DATABASE postgres
    SET "auto_explain.log_min_duration" TO '0';
ALTER ROLE monitoring IN DATABASE postgres
    SET "auto_explain.log_nested_statements" TO 'true';
ALTER ROLE monitoring IN DATABASE postgres
    SET "auto_explain.log_timing" TO 'true';
```

    
### 3. pg_stat_statements
 Это расширение ставится на двух уровнях
- на уравне сервера postgres -> файл postgresql.conf -> shared_preload_libraries
- на уровне базы
- расширение заполняет эту таблицу до 5000 после этого сбрасывается, поэтому надо чтобы инфа с переодичностью закидывалась в другую таблицу
  
```sql
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC;
calls -- сколько раз вызывался запрос
total_exec_time -- сколько съел ресурсов
SELECT pg_stat_statements_reset(); -- сброс
```
### 4. Для анализа логов 
- pgbadger
- pg_profile
# Версионность

```sql
SELECT txid_current();
SELECT xmin, xmax, cmin, cmax, ctid from pgbench_accounts;
```
### Чтобы посмотреть в страницу, необходимо поставить расширение pageinspect
### Поставим его и заглянем в страницу
```sql
CREATE EXTENSION pageinspect;
SELECT lp as tuple, t_xmin, t_xmax, t_field3, t_ctid FROM heap_page_items(get_raw_page('pgbench_accounts',1));
```

<table>
   <thead>
   <tr>
   <th>tuple</th>
   <th>t_xmin</th>
   <th>t_xmax</th>
   <th>t_field3</th>
   <th>t_ctid</th> 
   </tr>
   </thead>
<tr>
    <td>60</td>
   <td>570849</td>
   <td>0</td>
   <td>0</td>
   <td>(0,60)</td>
</tr>
   
   <tr>
    <td>61</td>
   <td>570849</td>
   <td>0</td>
   <td>0</td>
   <td>(0,61)</td>
</tr>
<tr>
    <td>62</td>
   <td>570847</td>
   <td>570849</td>
   <td>0</td>
   <td>(0,2)</td>
</tr>
</table>







