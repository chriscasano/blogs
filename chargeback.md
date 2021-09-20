# Chargeback

Here's a way you can create usage billing for applications that run queries on CockroachDB.  This can be useful in shared development environments which each application team is charged for the amount of resource utilization they incur.  Rates can be used to express the amount of memory, disk, network, etc are utilized during query execution.  The idea here is to take snapshots of the crdb_internal.node_statement_statistics table from each node in the cluster and reset the statistics once the snapshot is taken.  Unfortunately there's no way to automatically persist statement statistics today but this is an upcoming feature: https://github.com/cockroachdb/cockroach/issues/56219.  For now, the best thing to do is schedule a job that collects the statement stats on a periodic interval (<1 hour) and reset the statistics collection.

### Create Billing Database

Create a database with a Rate table and a table to record the statement statistics snapshots.

```sql
create database billing;

use billing;

create sequence batch owned by none;
select nextval('batch');

create table billing.rate
(
  id int primary key,
  unit string,
  rate decimal
);

insert into billing.rate
(id, unit, rate)
values
(1,'memory','.05'),
(2,'disk reads','.01'),
(3,'disk writes','.02'),
(4,'network','.001'),
(5,'cpu','0.00001')
;

-- show create crdb_internal.node_statement_statistics;
CREATE TABLE billing.usage (
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
full_scan BOOL not null,
batch int not null default currval('batch'),
insert_dt timestamp not null default now()::timestamp
);
```

### Run some workloads to generate some query usage

Run the following commands in one shell

```
cockroach workload init bank
cockroach workload run bank --max-rate=50
```

Run the following commands in different shell

```
cockroach workload init movr
cockroach workload run movr --max-rate=25
```

### At a given interval, take a snapshot of the statements statistics data and then reset the statistics;

In a new command shell, run the following commands when you want to get a snapshot of the statistical data.  

```
cockroach sql --insecure --port=26257 --database billing -e "insert into billing.usage (select *, (select last_value from batch), now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26259 --database billing -e "insert into billing.usage (select *, (select last_value from batch), now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26261 --database billing -e "insert into billing.usage (select *, (select last_value from batch), now()::TIMESTAMP from crdb_internal.node_statement_statistics);"
cockroach sql --insecure --port=26257 --database billing -e "select crdb_internal.reset_sql_stats(), nextval('batch');"
```

These statements can be scheduled to run on a periodic interval that falls below the time when statistics are automatically reset.  By default, statistics are reset every hour.

### Compute the usage cost by Application

Lastly, after you have some statement statistics persisted, you can run some report level queries which provide that amount of usage and cost incurred by each application that connects to the cluster.

The data from the statement statistics table keeps running statistics for a give unique SQL statement fingerprint.  That fingerprint will have the number of execution times, the average usage of resources (cpu, disk, ram, etc) and the variance of those resources as well.  The query below computes the amount of query executions for a SQL fingerprint, multiplied by average resource usage for a given timeframe.  The usage is then summarized by application.

```sql
select application_name, sum(cpu) as cpu, sum(cpu) * (select rate from billing.rate where unit = 'cpu') as cost
from
(
  select (u.count * u.service_lat_avg::decimal) as cpu, case when application_name like '$ internal%' then '$ internal' else application_name end as application_name
  from billing.usage u
)
group by application_name
;
```

Below is the output of the usage query above:

```
application_name   |           cpu            |            cost
-------------------+--------------------------+------------------------------
$ cockroach sql    |                 2.379611 |               0.00002379611
$ internal         | 105.39623999999999307846 | 0.0010539623999999999307846
bank               |   1984.63358200000077312 |    0.0198463358200000077312
movr               |  44.12923400000000044278 | 0.0004412923400000000044278
(4 rows)

Time: 5ms total (execution 4ms / network 0ms)
```
