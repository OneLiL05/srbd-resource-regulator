# СРБД - Регулятор ресурсів

## Налаштування та запуск

Запустимо PostgeSQL:

```sh
docker compose up -d
```

Підключимось до БД `chinook`, у якій вже будуть створені необхідні таблиці для виконання роботи (див. файл `seed.sql`):

```
docker exec -it srbd-postgres psql -U (chinook_low | chinook_high | postgres) -d chinook
```

## Конфігурація

Підключимось до БД під користувачем `postgres`:

```sh
docker exec -it srbd-postgres psql -U postgres -d chinook
```

Cтворимо додаткових користувачів, необхідних для виконання роботи:

```sql
CREATE ROLE chinook_low LOGIN PASSWORD 'StrongPass123';
CREATE ROLE chinook_high LOGIN PASSWORD 'StrongPass456';
```

Надамо їм привілеї над БД `chinook` та усіма її таблицями:

```sql
GRANT ALL PRIVILEGES ON DATABASE chinook TO chinook_low;
GRANT ALL PRIVILEGES ON DATABASE chinook TO chinook_high;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO chinook_low;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO chinook_high;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO chinook_low;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO chinook_high;
```

Встановимо обмеження для обох користувачів:

```sql
ALTER ROLE chinook_low SET statement_timeout = '10s';
ALTER ROLE chinook_low SET work_mem = '4MB';
ALTER ROLE chinook_low SET maintenance_work_mem = '16MB';
ALTER ROLE chinook_low CONNECTION LIMIT 5;

ALTER ROLE chinook_high SET statement_timeout = '3600s';
ALTER ROLE chinook_high SET work_mem = '512MB';
ALTER ROLE chinook_high SET maintenance_work_mem = '1GB'
ALTER ROLE chinook_high SET temp_file_limit = '10GB';
ALTER ROLE chinook_high SET hash_mem_multiplier = '2.0';
ALTER ROLE chinook_high CONNECTION LIMIT 20;
```

## Сценарії використання

Створення штучного навантаження одним із новостворених користувачів:

```sql
SELECT COUNT(*)
FROM "Customer" c
CROSS JOIN "Invoice" i
CROSS JOIN "InvoiceLine" il
CROSS JOIN "Track" t
CROSS JOIN "Album" a
CROSS JOIN "Genre" g;
```

Моніторинг використаних ресурсів:

```sql
SELECT
  userid::regrole AS username,
  COUNT(*) AS total_queries,
  SUM(calls) AS total_calls,
  SUM(total_exec_time)::NUMERIC(10,2) AS total_time_ms,
  AVG(mean_exec_time)::NUMERIC(10,2) AS avg_time_ms,
  SUM(rows) AS total_rows
FROM pg_stat_statements pss
JOIN pg_roles pr ON pss.userid = pr.oid
WHERE pr.rolname LIKE 'chinook%'
GROUP BY userid;
```
