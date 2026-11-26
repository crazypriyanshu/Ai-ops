# AIOps Lab Metrics Catalog

## 1. Payment Transaction Throughput (TPS)
* **PromQL:** `rate(pg_stat_database_xact_commit{datname="pay"}[1m])`
* **What it measures:** Committed SQL transactions per second on the `pay` database.
* **Data points:** Timestamped float values (1 sample every 15s) representing commits/sec.
* **Normal baseline:** ~200 TPS (driven by `pgbench -R 200`). Drops indicate lock contention or I/O stalls.

## 2. Failed / Rolled-Back Transactions
* **PromQL:** `rate(pg_stat_database_xact_rollback{datname="pay"}[1m])`
* **What it measures:** Aborted transactions per second.
* **Data points:** Timestamped float values (1 sample every 15s) representing rollbacks/sec.
* **Normal baseline:** 0.0/sec. Any sustained value > 0 signals application errors or deadlocks.

## 3. Database Write Intensity
* **PromQL:** `rate(pg_stat_database_tup_updated{datname="pay"}[1m])`
* **What it measures:** Table rows modified per second across `pay`.
* **Data points:** Timestamped float values (1 sample every 15s) representing rows updated/sec.
* **Normal baseline:** ~600 rows/sec (3 row updates per `pgbench` transaction).