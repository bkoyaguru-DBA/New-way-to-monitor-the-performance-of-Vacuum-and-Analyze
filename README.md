# New-way-to-monitor-the-performance-of-Vacuum-and-Analyze
PostgreSQL 18 will offer a new way to monitor the performance of Vacuum and Analyze

PostgreSQL 18: Vacuum and Analyze Execution Time Statistics
Prior to PostgreSQL 18, administrators had to rely on the `log_autovacuum_min_duration` parameter to capture vacuum and analyze execution times. This required enabling logging, parsing log files, and often using tools such as pgBadger to generate meaningful reports.

PostgreSQL 18 introduces new statistics columns in `pg_stat_all_tables` that store cumulative execution times for both manual and automatic vacuum/analyze operations. Combined with existing counters (`vacuum_count`, `autovacuum_count`, `analyze_count`, and `autoanalyze_count`), these metrics make it possible to calculate average execution times directly from SQL.

> **Note:** All execution times are stored in milliseconds.

---

## New Columns in `pg_stat_all_tables`

Let's examine the structure of `pg_stat_all_tables` and highlight the newly added columns.

```sql
postgres=# \d+ pg_stat_all_tables
```

Relevant columns:

| Column                   | Description                                        |
| ------------------------ | -------------------------------------------------- |
| `vacuum_count`           | Number of manual vacuum operations                 |
| `autovacuum_count`       | Number of autovacuum operations                    |
| `analyze_count`          | Number of manual analyze operations                |
| `autoanalyze_count`      | Number of autoanalyze operations                   |
| `total_vacuum_time`      | Total time spent in manual vacuum operations (ms)  |
| `total_autovacuum_time`  | Total time spent in autovacuum operations (ms)     |
| `total_analyze_time`     | Total time spent in manual analyze operations (ms) |
| `total_autoanalyze_time` | Total time spent in autoanalyze operations (ms)    |

These values are exposed through the following internal statistics functions:

```sql
pg_stat_get_total_vacuum_time(c.oid)
pg_stat_get_total_autovacuum_time(c.oid)
pg_stat_get_total_analyze_time(c.oid)
pg_stat_get_total_autoanalyze_time(c.oid)
```

---

# Generating Sample Data with pgbench

Initialize a sample `pgbench` database:

```bash
/usr/pgsql-18/bin/pgbench -h /tmp/ -d pgbench -i -s 100 -p 5460
```

Output:

```text
dropping old tables...
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 61.17 s
```

Connect to the database:

```sql
postgres=# \c pgbench
```

---

## Calculating Average Vacuum and Analyze Times

The following query calculates average execution times using the new PostgreSQL 18 statistics:

```sql
SELECT
    relname,
    vacuum_count,
    analyze_count,
    autovacuum_count,
    autoanalyze_count,
    ROUND(total_vacuum_time::numeric / NULLIF(vacuum_count,0), 2) AS avg_time_vacuum,
    ROUND(total_autovacuum_time::numeric / NULLIF(autovacuum_count,0), 2) AS avg_time_autovacuum,
    ROUND(total_analyze_time::numeric / NULLIF(analyze_count,0), 2) AS avg_time_analyze,
    ROUND(total_autoanalyze_time::numeric / NULLIF(autoanalyze_count,0), 2) AS avg_time_autoanalyze
FROM pg_stat_user_tables;
```

Sample output:

```text
     relname      | vacuum_count | analyze_count | autovacuum_count | autoanalyze_count | avg_time_vacuum | avg_time_autovacuum | avg_time_analyze | avg_time_autoanalyze
------------------+--------------+---------------+------------------+-------------------+-----------------+---------------------+------------------+----------------------
 pgbench_history  |            1 |             1 |                0 |                 0 |            1.00 |                     |             1.00 |
 pgbench_tellers  |            1 |             1 |                0 |                 0 |            1.00 |                     |             1.00 |
 pgbench_branches |            1 |             1 |                0 |                 0 |            1.00 |                     |             2.00 |
 pgbench_accounts |            1 |             1 |                0 |                 0 |           26.00 |                     |          6707.00 |
```

---

# Triggering an Autovacuum

To generate autovacuum activity, update a large number of rows:

```sql
UPDATE pgbench_accounts
SET filler = filler
WHERE abalance < 15;
```

Output:

```text
UPDATE 10000000
```

---

## Monitoring Vacuum Progress

Check autovacuum progress using:

```sql
SELECT * FROM pg_stat_progress_vacuum;
```

Example output:

```text
 pid  | datid | datname | relid |     phase     | heap_blks_total | heap_blks_scanned | heap_blks_vacuumed
------+-------+---------+-------+---------------+-----------------+-------------------+-------------------
 4849 | 16388 | pgbench | 49248 | scanning heap |          327869 |            190668 |                 0
```

---

## Viewing Updated Statistics

After autovacuum and autoanalyze complete, rerun the statistics query:

```sql
SELECT
    relname,
    vacuum_count,
    analyze_count,
    autovacuum_count,
    autoanalyze_count,
    ROUND(total_vacuum_time::numeric / NULLIF(vacuum_count,0), 2) AS avg_time_vacuum,
    ROUND(total_autovacuum_time::numeric / NULLIF(autovacuum_count,0), 2) AS avg_time_autovacuum,
    ROUND(total_analyze_time::numeric / NULLIF(analyze_count,0), 2) AS avg_time_analyze,
    ROUND(total_autoanalyze_time::numeric / NULLIF(autoanalyze_count,0), 2) AS avg_time_autoanalyze
FROM pg_stat_user_tables;
```

Example output:

```text
     relname      | vacuum_count | analyze_count | autovacuum_count | autoanalyze_count | avg_time_vacuum | avg_time_autovacuum | avg_time_analyze | avg_time_autoanalyze
------------------+--------------+---------------+------------------+-------------------+-----------------+---------------------+------------------+----------------------
 pgbench_accounts |            1 |             1 |                1 |                 1 |           11.00 |             7064.00 |          5468.00 |               806.00
 pgbench_branches |            1 |             1 |                0 |                 0 |           17.00 |                     |             1.00 |
 pgbench_history  |            1 |             1 |                0 |                 0 |            1.00 |                     |             1.00 |
 pgbench_tellers  |            1 |             1 |                0 |                 0 |            1.00 |                     |             1.00 |
```

---

# Useful Query for Monitoring Vacuum Performance

```sql
SELECT
    relname,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count,
    ROUND(total_vacuum_time::numeric / NULLIF(vacuum_count,0), 2) AS avg_time_vacuum_ms,
    ROUND(total_autovacuum_time::numeric / NULLIF(autovacuum_count,0), 2) AS avg_time_autovacuum_ms,
    ROUND(total_analyze_time::numeric / NULLIF(analyze_count,0), 2) AS avg_time_analyze_ms,
    ROUND(total_autoanalyze_time::numeric / NULLIF(autoanalyze_count,0), 2) AS avg_time_autoanalyze_ms
FROM pg_stat_user_tables
ORDER BY total_autovacuum_time DESC;
```

---

# Conclusion

PostgreSQL 18 significantly simplifies vacuum and analyze performance monitoring by exposing cumulative execution times directly in `pg_stat_all_tables`.

Benefits include:

* No need to enable `log_autovacuum_min_duration` solely for timing analysis.
* No requirement to parse PostgreSQL logs.
* Easier SQL-based monitoring and reporting.
* Ability to calculate average vacuum/analyze execution times per table.
* Better visibility into autovacuum performance without external tools.

This enhancement provides DBAs with a straightforward way to monitor and troubleshoot vacuum and analyze activity using native PostgreSQL statistics views.
