# BEGIN
### развернуть виртуальную машину любым удобным способом
### поставить на неё PostgreSQL 15 любым способом
### для мониторинга Postgresql, установил Prometheus, Grafana и PostgresExporter
![Иллюстрация к проекту](2024-12-23_10-58-32.png)

### настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
### Начальные настройки у нас такие
```sql
postgres=# show work_mem ;
 work_mem
----------
 4MB
(1 row)

postgres=# show effective_cache_size ;
 effective_cache_size
----------------------
 4GB
(1 row)

postgres=# show shared_buffers ;
 shared_buffers
----------------
 128MB
(1 row)

postgres=# show max_connections ;
 max_connections
-----------------
 100
(1 row)

postgres=# show maintenance_work_mem ;
 maintenance_work_mem
----------------------
 64MB
(1 row)

postgres=# show wal_buffers ;
 wal_buffers
-------------
 4MB
(1 row)

postgres=# show max_wal_size ;
 max_wal_size
--------------
 1GB
(1 row)

postgres=# show checkpoint_timeout ;
 checkpoint_timeout
--------------------
 5min
(1 row)

```
