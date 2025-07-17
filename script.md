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
- Уровень сервера postgresql.conf postgresql.auto.conf
- Уровень пользователя/роли
- Уровень Базы Данных
- Уровень сеанса
- Уровень процедуры/функции 
