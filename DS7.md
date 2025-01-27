#### Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
#### Установить на него PostgreSQL 15 с дефолтными настройками
#### Создать БД для тестов: выполнить pgbench -i postgres
#### Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
```sql
postgres@Ubuntu:~$  pgbench -c8 -P 6 -T 60 -U postgres postgres
starting vacuum...end.
progress: 6.0 s, 1078.5 tps, lat 7.388 ms stddev 6.324
progress: 12.0 s, 1176.3 tps, lat 6.794 ms stddev 5.828
progress: 18.0 s, 1150.5 tps, lat 6.947 ms stddev 6.120
progress: 24.0 s, 990.5 tps, lat 8.074 ms stddev 7.330
progress: 30.0 s, 1067.3 tps, lat 7.496 ms stddev 6.449
progress: 36.0 s, 1153.7 tps, lat 6.927 ms stddev 6.026
progress: 42.0 s, 1077.7 tps, lat 7.416 ms stddev 6.435
progress: 48.0 s, 1137.7 tps, lat 7.029 ms stddev 6.332
progress: 54.0 s, 1102.7 tps, lat 7.246 ms stddev 6.299
progress: 60.0 s, 1191.9 tps, lat 6.710 ms stddev 5.748
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 66768
latency average = 7.183 ms
latency stddev = 6.293 ms
tps = 1112.534045 (including connections establishing)
tps = 1112.567822 (excluding connections establishing)
```
#### Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
#### Протестировать заново
```sql
postgres@Ubuntu:~$  pgbench -c8 -P 6 -T 60 -U postgres postgres
starting vacuum...end.
progress: 6.0 s, 1192.8 tps, lat 6.669 ms stddev 5.460
progress: 12.0 s, 1234.3 tps, lat 6.475 ms stddev 5.247
progress: 18.0 s, 1253.5 tps, lat 6.375 ms stddev 5.365
progress: 24.0 s, 1262.3 tps, lat 6.335 ms stddev 5.695
progress: 30.0 s, 1248.0 tps, lat 6.407 ms stddev 5.540
progress: 36.0 s, 1243.5 tps, lat 6.431 ms stddev 5.590
progress: 42.0 s, 1262.2 tps, lat 6.332 ms stddev 5.322
progress: 48.0 s, 1245.0 tps, lat 6.419 ms stddev 5.377
progress: 54.0 s, 1256.9 tps, lat 6.361 ms stddev 5.307
progress: 60.0 s, 1246.8 tps, lat 6.412 ms stddev 5.371
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 74680
latency average = 6.422 ms
latency stddev = 5.435 ms
tps = 1243.971498 (including connections establishing)
tps = 1244.034881 (excluding connections establishing)
```
#### Что изменилось и почему?
#### Результаты улучшились. TPS возрос на 11%
* мы увеличили shared_buffers и сделали его 25% от общей памяти. Параметр shared_buffers определяет объем памяти, выделенной серверу для кэширования данных.
* параметр effective_cache_size влияет на предполагаемую стоимость сканирования индекса.
