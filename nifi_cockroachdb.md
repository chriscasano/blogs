# Scaling Ingest with Apache Nifi and CockroachDB


Ingesting data into CockroachDB using Apache NiFi?  This blog will help you get started.  The ingestion through NiFi is a standard, simple pattern just like other relational databases.  Especially if you are familiar with Postgres!  You can readily use the Postgres driver to build your first flow.  If you are not familiar with NiFi, there are plenty of places to learn including the QuickStart page and any of the Dzone articles or Github repos created by the legendary and ambitious flow master himself, Tim Spann.

The best processor to use for ingestion is the PutDatabaseRecord processor.  This uses JDBC for connectivity to CockroachDB and not the Bulk/IO interface.  Maybe we’ll explore that in a different blog.  If you are getting started, this tweets2crdb repo takes data from Twitter and loads it into a Cockroach DB if you need a flow reference point.

Scaling ingest for CockroachDB has similar feel to traditional RDBMs but has some differentiated scaling options.  The biggest takeaway is that you can ingest in parallel across all of the Cockroach nodes as opposed to going thru a single master writer or routing to different shards.  Architecturally, here are some thoughts on how you can optimize and scale your ingest.


#### 1) Use a load balancer (i.e. HAProxy)
This will ensure writes are distributed amongst all of the nodes in the CockroachDB cluster since writes can occur on any node.  As a transaction begins, the write itself will be routed to the correct node where the raft leader resides and the transaction will begin.  Fortunately, you can easily create a haproxy using CockroachDB which will create a round robin setup for your cluster.  Therefore, you should direct all of your ingestion traffic to the haproxy service  To do this in NiFi, you can set this up in database URL of the DBConnection Pool by providing the hostname/ip and port of the HAProxy service in the jdbc connection url (i.e. jdbc:postgresql://<ha-proxy-host>:<port>/<db-name>).



#### 2) Merge flowfiles (I.e. Create batches)

More than likely your initial flow is doing inserts one at a time.  In fact, the tweets2crdb repo I shared above does its ingest 1 at a time.  If you have one insert record in one flow file that is going into the PutDatabaseRecord process, you’re doing one insert at a time.  Instead try doing 100,1000 or 10000 inserts in a single flowfile.  Mileage and latency may vary.  To achieve this, utilize the MergeRecords processor which will combine flowfiles into batches.  You can put the MergeRecord processor right before the PutDatabaseRecord processor.  See image below.  Be sure your data is converted into an Avro format which is required for the PutDataRecord processor.


#### 3) Threads
Don’t forget to up the number of threads on your processors but do so wisely.  If you are batching records, you probably don’t need to increase the number of threads on the PutDatabaseRecord processor too much.  Once your flow is built, run the flow for an extended period of time with a lot of load to figure out where the optimization need to be made.  There’s a great article here about thread pools that’s worth reading.

#### 4) Connection Pools
A single connection pool should fulfill a majority of ingestion requirements.  Typically not much tuning is necessary to be done here. However, in geo-distributed Cockroach clusters, using one Connection Pool CockroachDB per region may make more sense  Yes, that would mean having a NiFi instance per region but those architectures should be given more thought depending on what you’re really trying to do.
Handle Failures & Retries - Errors happen.  Sometimes a database may be busy, a transaction gets aborted or an error occurs.  The PutDatabaseRecord process gracefully handles these errors and will pass them to either the failure queue or the retry queue.  If you do get a Primary Key failure there is a way of handling PK conflicts.

#### 5) Scale Horizontal
More than likely your throughput is not as high as Big Data ingest scenarios since CockroachDB typically handles more focused OLTP workloads.  But in the event it is, you can scale out NiFi in a cluster to increase throughput.  Additionally, you can add more CockroachDB nodes to increase the write throughput as well.  There are a number of workloads that come along with Cockroach DB that you can utilize to test write throughput.

If you're looking for more resources on how to do this, see this presentation I gave at a New York City meetup:

- [Slides](https://www.slideshare.net/ChrisCasano/scaling-writes-on-cockroachdb-with-apache-nifi)

- [Demonstration Video](https://youtu.be/Yw3i7og1RQ0)

- [Demonstration Repo](https://github.com/chriscasano/cockroach-nifi)


In conclusion, CockroachDB's motto is to make data easy.  Apache NiFi certainly makes data flow easy.  That's why the two of these technologies remind me of a solid peanut butter and jelly sandwich for your ingest!
