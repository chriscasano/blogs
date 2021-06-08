# Changed Data Capture from CockroachDB to ConfluentCloud

Here's a simple tutorial for sending data from CockroachDB directly to Confluent Cloud using CockroachDB Change Data Capture, which is typically refered to as a [Changefeed](https://www.cockroachlabs.com/docs/v20.2/create-changefeed.html).

## Setup Confluent Cloud

- [Create Free 30 Day ConfluentCloud Cluster](https://www.confluent.io/get-started/)
- [Add Confluent Cloud CLI](https://docs.confluent.io/ccloud-cli/current/install.html)

## Setup Your Kafka Cluster

#### Get Kafka Resource ID

The ID list here for your Kafka cluster will be needed in the steps below

```bash
ccloud kafka cluster list
```

#### Create API Keys

The API Key and API Secret are needed for creating the CockroachDB Changefeed

```bash
ccloud api-key create --resource <RESOURCE ID>
```

#### Get Kafka End Point

The end point is needed to connect the Changefeed to Kafka

```bash
ccloud kafka cluster describe <RESOURCE ID>
```

#### Create Topic

```bash
ccloud kafka topic create demo_t --partitions 6
```

#### Start a Kafka Consumer to Verify Your Change Data Feed

```bash
ccloud kafka topic consume demo_t
```

## Setup CockroachDB

Open a new terminal window and leave the Kafka consumer one open for later

#### Create CockroachDB Table

```bash
cockroach sql ...
```

```sql
create table t (k int default unique_rowid() primary key, v string);
```

#### Create Changefeed

When creating the changefeed, notice that you'll use 'kafka://' instead of using https:// or SASL_SSL://.  Also, be sure to include your API Key and Secret in the Changefeed.  

```sql
CREATE CHANGEFEED FOR TABLE t INTO 'kafka://<CONFLUENT CLOUD URL>:9092?sasl_enabled=true&sasl_password=<API SECRET>&sasl_user=<API KEY>&tls_enabled=true&topic_prefix=demo_' WITH updated, key_in_value, format = json;
```

#### Insert Some Rows

```sql
insert into t (v) values ('one');
insert into t (v) values ('two');
insert into t (v) values ('three');
```

#### Verify data is showing up in your consumer app

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a3d8jj833iq3z5v1ttuu.png)
