# Make table and index creation faster in a multi-region CockroachDB environment

Do you want to make your schema and object creation faster in CockroachDB?  I was working with a customer the other day that had a multi-region CockroachDB cluster spun up in US West 2, US East 1 and EU West 2 in AWS.  Anytime they created an object, their DDL statements would take a few seconds because of all of the hops they must make to system tables for creating the objects.  By default, the system tables are replicated uniformly across the cluster and the leaseholders/RAFT leaders (leaders for reads/writes) are dispersed in the cluster as well.  In their case, creating all of their database objects took almost 30 minutes.

One way to optimize the creation of these database objects is to move all of the leaseholders for tables in the system database to a specific region.  In the same region you move the leaseholder to, you should run the DDL statements

So, if you want to run all of your DDL changes in US East 1, run this command to move the leaseholder of the system database to US East 1.

```sql
alter database system configure zone using lease_preferences = '[[+region=us-east-1]]';
```

This will move all of the leaseholders in the system database to be in US-East-1.   This change is not immediate, so do wait a few minutes.  Then check the ranges of the system tables to see if their leaseholders were moved to us-east-1.  Here are some examples to test if the leaseholders moved:

```sql
show ranges from table system.users;
show ranges from table system.jobs;
show ranges from table system.zone;
...
```

```bash
root@localhost:26257/postgres> show ranges from table system.users;
  start_key | end_key | range_id | range_size_mb | lease_holder |           lease_holder_locality            | replicas |                                                            replica_localities
------------+---------+----------+---------------+--------------+--------------------------------------------+----------+-------------------------------------------------------------------------------------------------------------------------------------------
  NULL      | NULL    |        6 |      0.129162 |            2 | cloud=aws,region=us-east-1,zone=us-east-1b | {1,2,3}  | {"cloud=aws,region=us-west-2,zone=us-west-2b","cloud=aws,region=us-east-1,zone=us-east-1b","cloud=aws,region=eu-west-1,zone=eu-west-1b"}
(1 row)
```

Once the leaseholders have moved, try running your DDL statements and see if the creation time improves.

***This was before...***

```sql
CREATE TABLE CHRIS (PK INT PRIMARY KEY);

CREATE TABLE

Time: 1.905s total (execution 1.905s / network 0.000s)
```

***This was after...***

```sql
CREATE TABLE CHRIS (PK INT PRIMARY KEY);

CREATE TABLE

Time: 284ms total (execution 284ms / network 0ms)
```

I'm not a fan of changing system database stuff but sometimes this helps for setting up tests or recreating environments.  Just as easily it is to move the leaseholders into one region, you can always set it back to the default by running:

```sql
alter database system configure zone using lease_preferences = '[]';
```
