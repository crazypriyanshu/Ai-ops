Using 5 services to build a complete closed-loop observability pipeline: 
one service creates workload, 
one processes it, 
two translate internal telemetry, and 
one collects and stores the data.

1. postgres - The Core Application Database - real system doing real work so that metrics actually change over time.
It runs a PostgreSQL 16 database instance named pay (simulating a payment backend).

`track_io_timing=on` and `shared_preload_libraries=pg_stat_statements`: By default, databases hide low-level disk wait times and per-query latency to save overhead. These startup flags turn on deep internal instrumentation so we can later detect slow queries or disk bottlenecks.

volumes: ["pgdata:/var/lib/postgresql/data"]: Containers are temporary; if we delete a container, its files vanish. This creates a persistent virtual hard drive (pgdata) so your database survives restarts.

2. loadgen - Synthetic Traffic Generator - test anomaly detection, we need steady, predictable baseline traffic.
It runs pgbench (PostgreSQL's official benchmarking tool) to continuously read and write fake payment transactions against  postgres container.
`pgbench -i -s 20`: On first boot, it generates 2,000,000 rows of dummy account data (-s 20 means 20 × 100,000 rows) so queries have realistic table sizes to scan.

`while true; do pgbench -c 8 -j 2 -T 300 -R 200` ...: Runs an infinite loop simulating 8 concurrent users (-c 8) executing 200 transactions per second (-R 200).

3. postgres_exporter - The Database Translator : PostgreSQL speaks SQL and tracks its health in internal system tables (pg_stat_database, pg_locks), whereas Prometheus only understands HTTP text metrics.

Every time Prometheus scrapes it on port 9187, postgres_exporter runs SQL queries against postgres, converts the results into Prometheus's text format (e.g., pg_stat_database_xact_commit), and serves them over HTTP.

`DATA_SOURCE_NAME: "postgresql://postgres:lab@postgres:5432/pay?sslmode=disable"`: Notice @postgres:5432. Because Docker Compose puts all services on a private virtual network, postgres_exporter finds the database using its service name (postgres) as a hostname.

4. node_exporter — The Infrastructure / Hardware Translator : In production outages, a database often slows down not because of bad SQL, but because the underlying server ran out of CPU, RAM, or disk I/O bandwidth. AIOps requires correlating application metrics with hardware metrics.
It reads low-level Linux kernel counters (/proc and /sys) to measure CPU utilization across all 16 cores, memory allocation, disk read/write latency, and network traffic, exposing them on port 9100.

`pid: host and volumes: ["/:/host:ro"]`: Normally, a container is blind to the machine running it. These settings allow node_exporter to peek outside its container walls (read-only) to measure the actual host environment rather than just itself.

5. prometheus - The Central Time-Series Brain : Exporters (node_exporter and postgres_exporter) do not store history—they only show a live snapshot of right now when you visit their /metrics URL. To graph trends or train ML models, you need historical storage.

What it does: It is a time-series database and scraping engine. Every 5 seconds (defined in prometheus.yml), it sends HTTP GET requests to node_exporter:9100/metrics and postgres_exporter:9187/metrics, timestamps every number, and writes them to disk.


`--storage.tsdb.retention.time=14d`: Automatically manages disk space by keeping 14 days of historical telemetry in the promdata volume.

`ports: ["9090:9090"]`: Exposes the web UI and HTTP API to your Windows host at http://localhost:9090 so we can query the collected data with PromQL.
