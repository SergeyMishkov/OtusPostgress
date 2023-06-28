1. Развернул ВМ в Яндекс-облаке и подключился командой:
ssh -i ~/.ssh/yc_key otus@84.201.160.23

2. Установил постгрес 15 командой:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
Кластер стартовал:
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

8. На сайте https://pgtune.leopard.in.ua/#/ рекомендовано для настроек используемого мной сервера применить настройки:
max_connections = 20
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 13107kB
min_wal_size = 1GB
max_wal_size = 4GB

5. Получаем значения сейчас (по умолчанию) набором команд:
postgres=# show max_connections;
show shared_buffers;
show effective_cache_size;
show maintenance_work_mem;
show checkpoint_completion_target;
show wal_buffers;
show default_statistics_target;
show random_page_cost;
show effective_io_concurrency;
show work_mem;
show min_wal_size;
show max_wal_size;

6. Полученные значения:
max_connections              100
shared_buffers               128MB
effective_cache_size         4GB
maintenance_work_mem         64MB
checkpoint_completion_target 0.9
wal_buffers                  4MB
default_statistics_target    100
random_page_cost             4
effective_io_concurrency     1
work_mem                     4MB
min_wal_size                 80MB
max_wal_size                 1GB

Дополнительно из из предыдущих лекций для увеличения производительности (при уменьшении надежности) знаю, что следует поменять параметр synchronous_commit = off. Сейчас его значение равно "on".

7. Инициализирую pgbench 
postgres@otus-db-pg-vm-1:~$ pgbench -i postgres;

8. Запускаю pgbench:
postgres@otus-db-pg-vm-1:~$ pgbench -c 50 -j 2 -P 60 -T 600
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 515.4 tps, lat 96.623 ms stddev 88.978, 0 failed
progress: 120.0 s, 529.3 tps, lat 94.596 ms stddev 92.192, 0 failed
progress: 180.0 s, 472.3 tps, lat 105.827 ms stddev 113.656, 0 failed
progress: 240.0 s, 456.0 tps, lat 109.739 ms stddev 106.159, 0 failed
progress: 300.0 s, 558.2 tps, lat 89.576 ms stddev 83.477, 0 failed
progress: 360.0 s, 542.3 tps, lat 92.144 ms stddev 86.084, 0 failed
progress: 420.0 s, 491.1 tps, lat 101.901 ms stddev 103.916, 0 failed
progress: 480.0 s, 532.6 tps, lat 93.852 ms stddev 93.416, 0 failed
progress: 540.0 s, 536.8 tps, lat 93.146 ms stddev 90.849, 0 failed
progress: 600.0 s, 498.1 tps, lat 100.423 ms stddev 100.614, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 307973
number of failed transactions: 0 (0.000%)
latency average = 97.410 ms
latency stddev = 96.036 ms
initial connection time = 57.312 ms
tps = 513.243335 (without initial connection time)

9. Редактирую настройки в postgresql.conf:
otus@otus-db-pg-vm-1:~$ sudo nano /etc/postgresql/15/main/postgresql.conf

10. Изменяю параметры:
max_connections = 20                    # (change requires restart)
shared_buffers = 512MB                  # min 128kB
effective_cache_size = 1536MB
wal_buffers = 16MB
effective_io_concurrency = 2
work_mem = 13107kB
max_wal_size = 4GB
min_wal_size = 1GB
synchronous_commit = off

11. Перестартую сервис:
otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main restart

12. Перечитываю параметры + synchronous_commit. Они равны "ожидаемым".
postgres=# show max_connections;
show shared_buffers;
show effective_cache_size;
show maintenance_work_mem;
show checkpoint_completion_target;
show wal_buffers;
show default_statistics_target;
show random_page_cost;
show effective_io_concurrency;
show work_mem;
show min_wal_size;
show max_wal_size;
show synchronous_commit;

13. Перезапускаю pgbench:
postgres@otus-db-pg-vm-1:~$ pgbench -c 20 -j 2 -P 60 -T 600
pgbench (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 3307.2 tps, lat 6.044 ms stddev 2.848, 0 failed
progress: 120.0 s, 3314.0 tps, lat 6.034 ms stddev 2.868, 0 failed
progress: 180.0 s, 3274.0 tps, lat 6.108 ms stddev 2.832, 0 failed
progress: 240.0 s, 3246.5 tps, lat 6.160 ms stddev 2.855, 0 failed
progress: 300.0 s, 3248.4 tps, lat 6.156 ms stddev 2.805, 0 failed
progress: 360.0 s, 3335.6 tps, lat 5.995 ms stddev 2.789, 0 failed
progress: 420.0 s, 3339.9 tps, lat 5.987 ms stddev 2.776, 0 failed
progress: 480.0 s, 3277.1 tps, lat 6.102 ms stddev 2.784, 0 failed
progress: 540.0 s, 3195.4 tps, lat 6.258 ms stddev 2.831, 0 failed
progress: 600.0 s, 2964.7 tps, lat 6.745 ms stddev 3.044, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1950179
number of failed transactions: 0 (0.000%)
latency average = 6.153 ms
latency stddev = 2.850 ms
initial connection time = 23.433 ms
tps = 3250.167384 (without initial connection time)

14. Производительность существенно возросла.

15. Описание измененных мараметров:
max_connections - ограничивает число одновременных коннектов к базе. При его уменьшении соотвевенно уменьшается нагрузка на ресурсы.
shared_buffers - количество размещаемых данных в оперативной памяти (происходит гораздо более быстрый достп к данным, чем на диске)
effective_cache_size эффективный размер дискового кеша, доступного для одного запроса.
wal_buffers - объём разделяемой памяти, который будет использоваться для буферизации данных WAL, ещё не записанных на диск.
effective_io_concurrency - число параллельных операций ввода/вывода.
work_mem - базовый максимальный объём памяти, который будет использоваться во внутренних операциях при обработке запросов
max_wal_size - общий допустимый объем журнальных файлов
min_wal_size - сервер не удаляет файлы, пока они укладываются по объему в min_wal_size, а просто переименовывает их и использует заново
synchronous_commit = off - сервер не ждет пока записи из WAL сохранятся на диске, прежде чем сообщить клиенту об успешном завершении операции


Для выполнения ДЗ "интересно" поиграться параметрами, но для работы с ними "вживую" нужен более глубокий опыт их применения с помощью применения знаний курса, документации и методом проб и ошибок.