# CockroachCloud CDC to AWS Managed Kafka

- Create AWS VPC & Subnets
- Create AWS Managed Kafka (MKS)
- Edit Kafka Configuration
- Create CockroachCloud Cluster
- Setup PrivateLink between CockroahCloud and MKS
- Create and Test Changefeeds

## AWS Setup

[Create VPC](https://docs.aws.amazon.com/msk/latest/developerguide/create-vpc.html)

VPC ID: `vpc-02cdcc7d503037304`

[Create Subnets and Map Route Tables](https://docs.aws.amazon.com/msk/latest/developerguide/add-subnets.html)

Subnet 1: `subnet-0296c9875887da43f`
IP CIDR: 10.0.0.0/16
Route Table: `rtb-004b1ae347f321ebc`

Subnet 2: `subnet-04242a9385ac0142b`
IP CIDR: 10.0.1.0/16
Route Table:`rtb-015c96b220c6b2bb6`

Subnet 3: `subnet-03e5c728d7b1c6315`
IP CIDR: 10.0.2.0/16
Route Table:`rtb-015c96b220c6b2bb6`

VPC Security Group ID: `sg-02a7bdde17e4e4b5a`

## Create Kafka Cluster

[Create MKS cluster](https://docs.aws.amazon.com/msk/latest/developerguide/create-cluster.html)

Create clusterinfo.json
- Add subnets and security group to clusterinfo.json
- (optional) Remove encryption at rest items

Create Kafka Cluster

```
aws kafka create-cluster --cli-input-json fileb://clusterinfo.json
```

`{
  "ClusterArn": "arn:aws:kafka:us-east-1:541263489771:cluster/AWSKafkaTutorialCluster/f6faddf1-e0ef-4605-9bcd-62a698dc4692-3",
  "ClusterName": "AWSKafkaTutorialCluster",
  "State": "CREATING"
}
`

## Edit Kafka Cluster Configuration

Set a configuration for auto topic creation

```
auto.create.topics.enable=true
default.replication.factor=3
min.insync.replicas=2
num.io.threads=8
num.network.threads=5
num.partitions=1
num.replica.fetchers=2
replica.lag.time.max.ms=30000
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
socket.send.buffer.bytes=102400
unclean.leader.election.enable=true
zookeeper.session.timeout.ms=18000
```

## Create CockroachCloud Cluster

[Create CockroachCloud Cluster in a single AWS Region with 3 nodes](https://www.cockroachlabs.com/docs/cockroachcloud/create-your-cluster)

## Setup PrivateLink between Kafka VPC and CockroahCloud

[AWS PrivateLink Setup](
https://www.cockroachlabs.com/docs/cockroachcloud/network-authorization.html?_ga=2.256597000.2097469323.1624391348-1612776249.1600443819#create-an-aws-endpoint)

CockroachCloud Service: `com.amazonaws.vpce.us-east-1.vpce-svc-0decb1a4c9bacf685`

Create Endpoint in Kafka VPC
- In Security Group, add Inbound/Outbound rules for Kafka Brokers & CRDB
  - 9094 - Kafka Broker TLS
  - 9092 - Kafka Broker Plaintext
  - 2181 - Zookeeper
  - 26257 - CockroachDB


[Enable Private DNS](https://www.cockroachlabs.com/docs/cockroachcloud/network-authorization.html?_ga=2.256597000.2097469323.1624391348-1612776249.1600443819#enable-private-dns)

## Create Changefeeds

Shell into CockroachCloud

```
cockroach sql --url 'postgres://chris@casano-cdc-7h9.aws-us-east-1.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full&sslrootcert=/Users/chriscasano/.cockroach-certs/casano-cdc-ca.crt'
```

Create test table
```
create table test (i int default unique_rowid() primary key, t string);
```

Enable Changefeeds

```
set cluster setting kv.rangefeed.enabled = true;
```

Create Changefeeds

```
CREATE CHANGEFEED FOR TABLE test INTO 'kafka://b-1.awskafkatutorialcluste.4yynm6.c3.kafka.us-east-1.amazonaws.com:9094?tls_enabled=true&sasl_enabled=true&sasl_mechanism=SCRAM-SHA-512&sasl_user=chris&sasl_password=chris-secret' WITH updated, resolved;
```
