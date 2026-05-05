Проверим влияние различных конфигураций PostgreSQL в докер контейнерах. Тесты проведены на домашнем ноутбуке (7840hs, 16 RAM) под управлением Arch linux.

Для начала нужно создать Docker контейнер с БД, для этого создаем его из образа 
```
docker run --name postgres-container -e POSTGRES_PASSWORD=password -d -p 3005:5432 postgres
```
Затем нужен контейнер HammerDB и настраиваем его через CLI
```
dbset db pg

# Configure connection parameters
diset connection pg_host posthres-container
diset connection pg_port 3005               
diset connection pg_superuser postgres      # Your PostgreSQL superuser
diset connection pg_superuserpass password
diset connection pg_defaultdbase postgres

# Configure TPC-C benchmark parameters
diset tpcc pg_user postgres                 # User for benchmark tables
diset tpcc pg_pass password
diset tpcc pg_dbase tpcc_db                 # Database name for benchmark
diset tpcc pg_total_iterations 1000000      # Total transactions per user
diset tpcc pg_rampup 1                      # Ramp-up time in minutes
diset tpcc pg_duration 5                    # Test duration in minutes

# Set the workload and build the schema
dbset bm TPC-C
```

Можно приступать к тестированию
```
buildschema
loadscript
vucreate
vurun
vustatus
vudestroy
```

На базовой конфигурации без дополнительно установленных параметров, контейнер Postgresql под debian выдает следующий результат
```
Vuser 1:TEST RESULT : System achieved 9720 NOPM from 22368 PostgreSQL TPM
```

Теперь перезапустим контейнер с БД с другой конфигурацией, мой наилучший результат:
```
docker run --name postgres-container -e POSTGRES_PASSWORD=password -d -p 3005:5432 postgres \
-c shared_buffers='2560MB' \
  -c effective_cache_size='7680MB' \
  -c work_mem='10MB' \
  -c maintenance_work_mem='640MB' \
  -c wal_buffers='8MB' \
  -c max_connections='300' \
  -c min_wal_size='2GB' \
  -c max_wal_size='8GB' \
  -c checkpoint_completion_target='0.9' \
  -c random_page_cost='1.1' \
  -c autovacuum='on' \
  -c max_wal_senders='0' \
  -c wal_keep_size='0' \
  -c shared_preload_libraries='pg_stat_statements' \
  -c track_activity_query_size='2048' \
  -c pg_stat_statements.track='all' \
  -c log_min_duration_statement='1000' \
  -c synchronous_commit='off' \
  -c full_page_writes='off'
```

```
TEST RESULT : System achieved 35184 NOPM from 81187 PostgreSQL TPM
```

Выделение большего количества ресурсов принципиально увеличило производительность БД

https://github.com/Affirmat1ve/db_opt_singlepage/blob/main/hammerDB/result.md
