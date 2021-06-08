# Chargeback

Here's a way you can create usage billing for applications that run queries on CockroachDB.  Rates can be used to express the amount of memory, disk, network, etc are utilized during query execution.  The idea here is to take snapshots of the crdb_internal.node_statement_statistics table from each node in the cluster and reset the statistics once the snapshot is taken.  Unfortunately there's no way to reset the statistics on a schedule but this appears to be an upcoming feature: https://github.com/cockroachdb/cockroach/issues/64144

### Create Billing Database

Create a database with a Rate table and a table to record the statement statistics snapshots.

```sql
create database billing;

use billing;

create table billing.rate
(
  id int primary key,
  unit string,
  rate decimal
);
```

```sql
insert into billing.rate
(id, unit, rate)
values
(1,'memory','.05'),
(2,'disk reads','.01'),
(3,'disk writes','.02'),
(4,'network','.001'),
(5,'cpu','0.0')
;
```


```sql
-- show create crdb_internal.node_statement_statistics;
CREATE TABLE usage (
node_id INT8 NOT NULL,
application_name STRING NOT NULL,
flags STRING NOT NULL,
key STRING NOT NULL,
anonymized STRING NULL,
count INT8 NOT NULL,
first_attempt_count INT8 NOT NULL,
max_retries INT8 NOT NULL,
last_error STRING NULL,
rows_avg FLOAT8 NOT NULL,
rows_var FLOAT8 NOT NULL,
parse_lat_avg FLOAT8 NOT NULL,
parse_lat_var FLOAT8 NOT NULL,
plan_lat_avg FLOAT8 NOT NULL,
plan_lat_var FLOAT8 NOT NULL,
run_lat_avg FLOAT8 NOT NULL,
run_lat_var FLOAT8 NOT NULL,
service_lat_avg FLOAT8 NOT NULL,
service_lat_var FLOAT8 NOT NULL,
overhead_lat_avg FLOAT8 NOT NULL,
overhead_lat_var FLOAT8 NOT NULL,
bytes_read_avg FLOAT8 NOT NULL,
bytes_read_var FLOAT8 NOT NULL,
rows_read_avg FLOAT8 NOT NULL,
rows_read_var FLOAT8 NOT NULL,
network_bytes_avg FLOAT8 NULL,
network_bytes_var FLOAT8 NULL,
network_msgs_avg FLOAT8 NULL,
network_msgs_var FLOAT8 NULL,
max_mem_usage_avg FLOAT8 NULL,
max_mem_usage_var FLOAT8 NULL,
max_disk_usage_avg FLOAT8 NULL,
max_disk_usage_var FLOAT8 NULL,
contention_time_avg FLOAT8 NULL,
contention_time_var FLOAT8 NULL,
implicit_txn BOOL NOT NULL,
insert_dt timestamp not null
);
```

### Run a workload to generate some queries

```
cockroach workload init bank
cockroach workload init movr

cockroach workload run bank --max-rate=50
cockroach workload init movr --max-rate=50
```


### At a given interval, take a snapshot of the statements statistics data

```
cockroach sql --insecure --port=26257 -e "insert into billing.usage (select *, now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26259 -e "insert into billing.usage (select *, now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26261 -e "insert into billing.usage (select *, now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26263 -e "insert into billing.usage (select *, now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26265 -e "insert into billing.usage (select *, now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
```


###  Reset node_statement_statistics

???
