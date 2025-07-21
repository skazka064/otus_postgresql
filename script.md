## psql https://www.interdb.jp/pg/
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
## Настройка Postgresql
### Уровни настройки
- Уровень сервера **(postgresql.conf postgresql.auto.conf)**
- Уровень пользователя/роли **(ALTER ROLE postgres SET work_mem TO '256MB')**
- Уровень Базы Данных **(ALTER DATABASE postgres SET work_mem TO '128MB')**
- Уровень сеанса **(set work_mem ='256MB')**
- Уровень процедуры/функции **(CREATE FUNCTION .... SET configuration_parameter TO value)**
### Посмотреть настройки сеанса
- show all
- explain (analyze, settings) select * from pgbench_accounts; **Посмотреть, какие настройки скручены.**
### Настройка производительности
1. Настраиваем сервер сограсно типу нагрузки ( https://pgtune.leopard.in.ua/ )
2. Создаем тестовую базу ( pgbench -i -s 10 ) (pgbench -c 10 -j 1 -t 5000)
3. Центральный процессор
   - max_worker_processes = 28
   - max_parallel_workers_per_gather = 4
   - max_parallel_workers = 28
   - max_parallel_maintenance_workers = 4
   - top (смотрим параметр ID - сколько процессорных ресурсов находятся в резерве, должно быть не менее 20%)
4. Память
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

6. Сеть (tc -s -d qdisc ls dev enp0s3)
