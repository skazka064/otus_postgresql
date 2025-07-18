## psql
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
   ![Memory]([https://github.com/skazka064/otus_postgresql/tree/main/img/pg_memory.png])
        - ps auxf | grep postgres; ps fp $(pgrep post)
6. Сеть (tc -s -d qdisc ls dev enp0s3)
