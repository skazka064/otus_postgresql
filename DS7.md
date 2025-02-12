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
* мы увеличили shared_buffers и сделали его 25% от общей памяти. Параметр shared_buffers определяет объем памяти, выделенной серверу для кэширования данных. По ДЗ RAM=4G. Мы установили 1G, что соответствует рекомендациям. 
* Уменьшили параметр effective_cache_size с 4G до 3G. Он оценивает, сколько памяти доступно для кэширования диска ОС и в самой базе данных. Планировщик запросов Postgresql решает, зафиксировано ли это в оперативной памяти или нет.Вообщем этот параметр сообщает Postgresql, сколько памяти доступно для кэширования его файлов. Если значение высокое, то Potgresql будет оценивать соединения вложенных циклов с индексным сканированием дешевле, поскольку он предполагает, что индекс скорее всего будет кэширован. Сканирование индекса, скорее всего будет использоваться для более высоких значений, в противном случая, если значение низкое, будет использоваться последовательное сканирование. Рекомендуется устанавливать effective_cache_size на уровне 50% от общего объема оперативной памяти машины. По заданию ДЗ ОЗУ=4G. Мы поставили 3G, хотя по рекомендации надо было ставить 2G.
* maintenance_work_mem установили 512MB. т.е. увеличили с 64MB. Этот параметр по сути определяет максимальный объем памяти, который будет использоваться операциями обслуживания. Что-бы рассчитать подходящее значение, можно воспользоваться следующей формулой ( по рекомендации AWS) maintenance_work_mem=(total_memory-shared_buffers)/(max_connection\*5)=(4-1)/(40\*5)=15MB. Чем больше памяти, тем большее значение можно выбрать, и чем больше количество подключений, тем меньше можно брать maintenance_work_mem , потому что при запуске автовакуума эта память может быть выделена до autovacuum_max_workers раз, поэтому слишком большое значение не рекомендуется устанавливать. Можно этот процесс так же контролировать, отдельно устанавливая autovacuum_work_mem. Но только одна из операций VACUUM, CREATE INDEX, ALTER TABLE, ADD FOREIGN KEY может быть выполнена сеансом базы данных за раз , а обычно не выполняется много из них одновременно, то можно безопасно установить это значение значительно больше work_mem. Большие настройки могут улучшить производительность очистки и восстановления БД. Поэтому, выставляем это значение в зависимости от контекста.
* checkpoint_completion_target, по заданию равен 0.9. Этот параметр указывает цель завершения контрольной точки, как долю от общего времени между контрольными точками. Значение по умолчанию 0.9, что распределяет контрольную точку почти по всему дотупному интервалу, обеспечивая довольно постоянную нагрузку ввода-вывода, а так же оставляет некоторое время для накладных расходов на завершение контрольной точки. Уменьшать этот параметр не рекомендуется, поскольку приведет к более быстрому завершению КТ. А это, в свою очередь приведет к более высокой скорости ввода-вывода во время КТ, за которой следует период меньшего ввода-вывода между завершением КТ и следующей запланированной. контрольной точкой
