# Getting started with Prometheus and TimescaleDB

If you ever needed a monitoring solution for your infrastructure than you've
probably heard about [Prometheus][]. Prometheus is an
open-source and community driven monitoring system which is quite simple to
start with. What makes Prometheus awesome is its unapologetic approach to
solving monitoring in a simple and straightforward way. Their philosophy is to
do one thing, and do it well. This is reflected in, e.g., the design of the
PromQL language.

However this philosophy can also be limiting. To their credit, the developers
of Prometheus foresaw that their product is opinionated, and built in extensibility
to allow other systems to improve on it. In turn, Prometheus users often look to
other systems as a way to augment their monitoring setup.

This is where TimescaleDB and PostgreSQL come in. In particular, Prometheus
users often turn to TimescaleDB and PostgreSQL for a few reasons:

- Scalable, durable long-term data storage
- Operational ease with a broad tooling ecosystem
- Query power and flexibility with SQL

By using Prometheus and TimescaleDB together, you can combine the simplicity of
Prometheus with the reliability, power, and flexibility of TimescaleDB, and
pick the approach that makes most sense for the task at hand. For instance, you
can use either PromQL or full SQL for your queries, or both.

The purpose of this tutorial is to demonstrate how you can integrate Prometheus
with TimescaleDB and get the best of both worlds. But let’s first try to better
understand why using TimescaleDB can help add functionality to Prometheus and
make it even better.

>:TIP: If you need to understand why TimescaleDB is better than PostgreSQL for
storing metrics data you can read [here][timescaledb vs]

## Why use TimescaleDB with Prometheus

### Long term data storage

In order to keep Prometheus simple and easy to operate, its creators
intentionally left out some of the scaling features one would normally expect.
The data in Prometheus is stored locally within the instance and is not
replicated. Having both compute and data storage on one node may make it easier
to operate, but also makes it harder to scale and ensure high availability.

As a result, Prometheus is not designed to be a long-term metrics store. From
the [documentation][prometheus docs]:


>[Prometheus][] is not arbitrarily scalable or durable in the face of disk or node outages and should thus be treated as more of an ephemeral sliding window of recent data.

On the other hand, TimescaleDB can easily handle terabytes of data, and
supports high availability and replication, making it a good fit for long term
data storage. In addition, it provides advanced capabilities and features, such
as full SQL, joins and replication, which is simply not available in Prometheus.

How it works: All metrics that are recorded into Prometheus are first written
to the local node, and then written to TimescaleDB. This means that all of your
metrics are immediately backed up, so that any disk failure on a Prometheus
node will be less painful.

TimescaleDB can also store other types of data (metrics, time-series,
relational), allowing you to centralize your monitoring data from different
sources and simplify your stack. You can even join those different types of data
and add context to your monitoring data for richer analysis.

### Operational ease

There are [several][prometheus lts] long-term storage options for Prometheus.
However, the challenge with all of them is that they will likely require your
team to operate and manage yet another system.

TimescaleDB is an exception. TimescaleDB looks and operates like PostgreSQL,
which means that if you are already operating a PostgreSQL instance, you can
re-use that knowledge in operating a TimescaleDB node or cluster. It also means
that you can also use any of the existing tooling in the broad PostgreSQL
ecosystem.

The only way to scale Prometheus is by [federation][]. However, there are cases
where federation is not a good fit: for example, when copying large amounts of
data from multiple Prometheus instances to be handled by a single machine.
This is can result in poor performance, decreased reliability
(an additional point of failure), and loss of some data. These are all the
problems you can `outsource` to TimescaleDB.

### SQL query power

PromQL is the Prometheus native query language. It’s a very powerful and
expressive query language that allows you to easily slice and dice metrics data
and apply variety of monitoring-specific functions.

However, there may be times where you need greater query flexibility and power
than what PromQL provides. For example, trying to cross-correlate metrics with
events and incidents that occurred, running more granular queries for active
troubleshooting, or applying machine learning or other forms of deeper analysis
on metrics.

This is where the full SQL support of TimescaleDB can be of great help. With
TimescaleDB, you can apply the full breadth of SQL to your Prometheus data,
joining your metrics data with any other data you might have, and run more
powerful queries (Some examples are shared later in this tutorial).

## How to use TimescaleDB with Prometheus [](how-to)

To connect TimescaleDB and PostgreSQL to Prometheus, there are two components:

1. The `Prometheus PostgreSQL Adapter`
2. A PostgreSQL database with the `pg_prometheus` and `timescaledb` extensions


At a high-level, here is how it works:

<img class="main-content__illustration" src="http://assets.iobeam.com/images/docs/collecting-metrics.jpg" alt="Prometheus TimescaleDB Adapter"/>


All the data is first collected into Prometheus. Then Prometheus forwards it
to the configured [Prometheus PostgreSQL Adapter][postgresql adapter] and
then further towards the database with the [pg_prometheus extension][pg_prometheus] and finally to TimescaleDB.

### Prometheus PostgreSQL Adapter - the remote storage adapter

The adapter is basically a translation proxy that Prometheus uses for reading and writing data into TimescaleDB/PostgreSQL. Whenever Prometheus scrapes some service for metrics it will send the data to the adapter who is responsible for writing the data to the database. Because the data from Prometheus arrives as a Protobuf, it needs to be first deserialized and then converted into the [Prometheus native format][] before it is inserted into the database.

The adapter has a dependency on the pg_prometheus PostgreSQL extension, which takes care of writing the data in most optimal format for storage and querying within TimescaleDB/PostgreSQL.

### Pg_Prometheus

In order to slim down the adapter and enable seamless integration with the
database, we decided to build a PostgreSQL extension. The extension translates
from the [Prometheus data model][Prometheus native format]
into a compact SQL model that is stored efficiently and is easy to query.

To do this, Pg_Prometheus stores data in two tables in a “normalized format”,
where labels are stored in a `labels` table and metric values in a `values`
table. Normalizing the data across those two tables helps with saving disk
space, as labels can be long and repetitive.

The `values` table schema:
```sql
 Column   |           Type           | Modifiers
-----------+--------------------------+-----------
time      | timestamp with time zone |
value     | double precision         |
labels_id | integer                  |
```

The `labels` table schema:
```sql
  Column    |  Type   |                          Modifiers
-------------+---------+-------------------------------------------------------------
id          | integer | not null default nextval('metrics_labels_id_seq'::regclass)
metric_name | text    | not null
labels      | jsonb   |
```

But, to make it simple to query the data, we’ve defined a `metrics` view, that provides a unified view across all your data:

```sql
Column |           Type           | Collation | Nullable | Default
--------+--------------------------+-----------+----------+---------
sample | prom_sample              |           |          |
time   | timestamp with time zone |           |          |
name   | text                     |           |          |
value  | double precision         |           |          |
labels | jsonb                    |           |          |

```

(Note: There is also an option to store data in a raw format. You can find more information [here][pg_prometheus].)

## Let's try it out!

The easiest way to get started is by using Docker images. Make sure you have
Docker installed on your local machine (installation instructions
[here][docker]).

(Please note that the instructions below are only for trying out things locally
and should not be used to set up a production environment.)

### Create a bridge network

Let’s first create a bridge network that all our containers can connect to and
talk to each other.
```bash
docker network create -d bridge prometheus_timescale_network
```

### Spin up Pg_Prometheus

Now, let’s spin up a database with the [Pg_Prometheus extension][pg_prometheus]. This is where we are
going to store all our metrics that Prometheus scrapes for. For this, we will
use a PostgreSQL docker image that has both Pg_Prometheus and TimescaleDB
extensions installed: https://hub.docker.com/r/timescale/pg_prometheus.

```bash
docker run --network prometheus_timescale_network  --name pg_prometheus -d -p 5432:5432 timescale/pg_prometheus:master postgres \
     -csynchronous_commit=off
```

Note that this image inherits from the official PostgreSQL image and so all
options documented there are applicable to this image as well. This is
especially important for users that wish to persist data outside of docker
volumes is the PGDATA environment variable and accompanying volume mount (visit https://hub.docker.com/_/postgres/ for available configuration options)

### Spin up the Prometheus PostgreSQL adapter

Since we have the database up and running, let’s spin up a [Prometheus PostgreSQL adapter][postgresql adapter]. A [docker image][] is available on Docker Hub:

```bash
docker run --network prometheus_timescale_network --name prometheus_postgresql_adapter -d -p 9201:9201 \
timescale/prometheus-postgresql-adapter:master \
-pg.host=pg_prometheus \
-pg.prometheus-log-samples
```

### Start collecting metrics

We need a service that will expose metrics to Prometheus. For this tutorial,
we’ll start [Node Exporter][] which
is a Prometheus exporter for hardware and OS metrics.

```bash
docker run --network prometheus_timescale_network --name node_exporter -p 9100:9100 quay.io/prometheus/node-exporter
```

Once the Node Exporter is running, you can verify that system metrics is being exported by visiting
`http://localhost:9100/metrics`. Prometheus will scrape `/metrics` endpoint to get metrics.

### Spin up Prometheus

Finally we need to spin up Prometheus. We need to make sure that Prometheus
config file `prometheus.yml` is pointing to our adapter and that we’ve properly
set scrape config target to point to our Node Exporter instance.

Below is very basic `prometheus.yml` that we can use for this tutorial (click
[here][first steps] to read more on
Prometheus configuration)
```
global:
 scrape_interval:     10s
 evaluation_interval: 10s
scrape_configs:
 - job_name: prometheus
   static_configs:
     - targets: ['node_exporter:9100']
remote_write:
 - url: "http://prometheus_postgresql_adapter:9201/write"
remote_read:
 - url: "http://prometheus_postgresql_adapter:9201/read"
```


```bash
docker run --network prometheus_timescale_network -p 9090:9090 -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
      prom/prometheus
```

To avoid running each docker container separately, here is the `docker-compose.yml` that will
spin up all the docker containers together (Make sure you have `prometheus.yml` config file in the same folder as `docker-compose.yml`)

```
version: '2.1'
services:
 pg_prometheus:
   image: timescale/pg_prometheus:master
   command: -c synchronous_commit=OFF
   container_name: pg_prometheus
   healthcheck:
     test: ["CMD-SHELL", "pg_isready -U postgres"]
     interval: 1s
     timeout: 5s  
     retries: 10
 prometheus_postgresql_adapter:
   image: timescale/prometheus-postgresql-adapter:master
   ports:
     - "9201:9201"
   depends_on:
     pg_prometheus:
       condition: service_healthy
   command: "-pg.host=pg_prometheus -pg.prometheus-log-samples"
 node_exporter:
   image: quay.io/prometheus/node-exporter
   ports:
     - "9100:9100"
 prometheus:
   image: prom/prometheus
   ports:
     - "9090:9090"
   volumes:
     - ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml
```

Fire it up with `docker-compose up`


### Run queries! [](run-queries)

As specified in the configuration above, the Prometheus will scrape Node
Exporter every 10s and metrics will end up in both Prometheus local storage and
TimescaleDB. Now we can run few queries to make sure that data is coming in.

To run PromQL queries go to http://localhost:9090/graph. SQL queries you can
run from your favorite SQL tool or from the Pg_Prometheus docker container:

```bash
docker exec -it pg_prometheus bash
```

Once in the container use psql `psql postgres postgres` to connect to
TimescaleDB and run SQL queries. Below you can find some example queries.

**CPU frequency values for core “0” for the last 5 minutes (gauge):**

PromQL:
```
node_cpu_frequency_hertz{cpu="0"}[5m]
```

SQL:
```sql
SELECT time, value
FROM metrics
WHERE name='node_cpu_frequency_hertz' AND labels->>'cpu' = '0' AND
      time > NOW() - interval '5 min'
```


**Max CPU frequency by CPU core in the last 5 minutes:**

PromQL:
```
max(max_over_time(node_cpu_frequency_hertz[5m])) by (cpu)
```

SQL:
```sql
SELECT labels->>'cpu', MAX(value)
FROM metrics
WHERE name='node_cpu_frequency_hertz' AND
    time > NOW() - interval '5 min'
GROUP BY labels->>'cpu';
```


**Get average system load and used memory grouped in 5 minute buckets for the last 24 hours:**

PromQL: not possible (only can return one value)

SQL:
```sql
SELECT time_bucket('5 minutes', time) AS five_min_bucket, name, avg(value)
FROM metrics
WHERE (name='node_load5' OR name='node_memory_Active_bytes') AND
      time > NOW() - interval '1 day'
GROUP BY five_min_bucket,name
ORDER BY five_min_bucket;
```

Note that above SQL query is taking advantage of the `time_bucket` function only available in TimescaleDB.

**Check whether system performance was impacted by a Linux kernel update:**

If we have an additional table in TimescaleDB containing information about Linux kernel updates per host we could easily join that data with metrics from Prometheus and figure out if maybe a kernel patch decreases system performances ([for example][]).

PromQL: not a good fit because you would need metadata in Prometheus

SQL:

Clearly, enriching and correlated your data from different sources is pretty simple with TimescaleDB; it’s just a plain old `JOIN` statement. An example query
could look like:

```sql
SELECT time_bucket('1 hour', m.time) AS hour_bucket, 
       m.labels->>'host', h.kernel_updated, AVG(value)
FROM metrics m LEFT JOIN hosts h on h.host = m.labels->>'host' 
AND  time_bucket('1 hour', m.time) = time_bucket('1 hour', h.kernel_updated) 
WHERE m.name='node_load5' AND m.time > NOW() - interval '7 days'
GROUP BY hour_bucket, m.labels->>'host', h.kernel_updated
ORDER BY hour_bucket;
```

Integrating your Prometheus instance(s) with TimescaleDB is simple and risk-free. All your data will still be available in Prometheus and you can continue using all the Prometheus features as before. TimescaleDB’s integration just gives you scalability, durability, and greater query power, opening up a door for a completely new perspective on your monitoring data.


[Prometheus]: https://prometheus.io/
[timescaledb vs]: /introduction/timescaledb-vs-postgres
[prometheus docs]: https://prometheus.io/docs/prometheus/2.3/storage/
[prometheus lts]: https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage
[federation]: https://prometheus.io/docs/prometheus/latest/federation/
[postgresql adapter]: https://github.com/timescale/prometheus-postgresql-adapter
[pg_prometheus]: https://github.com/timescale/pg_prometheus
[Prometheus native format]: https://prometheus.io/docs/instrumenting/exposition_formats/
[docker]: https://docs.docker.com/install
[docker image]: https://hub.docker.com/r/timescale/prometheus-postgresql-adapter
[Node Exporter]: https://github.com/prometheus/node_exporter
[first steps]: https://prometheus.io/docs/introduction/first_steps/
[for example]: https://www.zdnet.com/article/linux-meltdown-patch-up-to-800-percent-cpu-overhead-netflix-tests-show/
