Getting Started
----------------------------------------------

### Introduction

<!-- TOC titleSize:3 tabSpaces:2 depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 skip:0 title:1 charForUnorderedList:* -->
### Table of Contents
* [Introduction](#introduction)
* [Requirements](#requirements)
  * [Use Case Requirements](#use-case-requirements)
  * [Performance Requirements](#performance-requirements)
  * [Required SLA](#required-sla)
* [Creating the Data Model](#creating-the-data-model)
  * [Design and Data Model](#design-and-data-model)
  * [Conceptual Data Model](#conceptual-data-model)
  * [Application Workflow](#application-workflow)
  * [Queries](#queries)
  * [Logical Data Model](#logical-data-model)
  * [Physical Data Model](#physical-data-model)
* [Deploying the App](#deploying-the-app)
  * [Prerequisites](#prerequisites)
  * [Download the code](#download-the-code)
  * [Start the pet collar simulation](#start-the-pet-collar-simulation)
  * [Start the REST API service](#start-the-rest-api-service)
* [Running the Application](#running-the-application)
* [REST API Example](#rest-api-example)
  * [Read owner data](#read-owner-data)
* [Code Structure and Implementation](#code-structure-and-implementation)
* [API examples](#api-examples)
  * [List the owner's pets](#list-the-owners-pets)
  * [List each specific pet's sensor](#list-each-specific-pets-sensor)
  * [Review the data from a specific sensor](#review-the-data-from-a-specific-sensor)
  * [Read the pet's daily average per sensor use](#read-the-pets-daily-average-per-sensor-use)
* [Additional Resources](#additional-resources)
<!-- /TOC -->

This guide is intended for users who have a basic understanding of ScyllaDB and APIs.

It demonstrates how to implement and configure ScyllaDB as the backend datastore for an IoT app called CarePet.

CarePet collects and analyzes data from sensors attached to a pet's collar and monitors the pet's health and activity.

This guide can be used with minimal changes for any IoT like application.

The different stages of development are as follows:
1. [Requirements](#gathering-requirements)
2. [Creating the data model](#creating-the-data-model)
3. [Cluster sizing](#logical-data-model)
4. Hardware needed to match the requirements
5. [Running the application](#using-the-application).

### Requirements

#### Use Case Requirements

Each pet collar includes sensors that report four different measurements:
* Temperature
* Pulse
* Location
* Respiration

The collar reads the sensor's data once a second and sends measurements directly to the app.

#### Performance Requirements

The application has two parts:

-   **Sensors**: Writes to the database, throughput sensitive.
-   **Backend dashboard**: Reads from the database, latency-sensitive.

For this example, we assume 99% writes (sensors) and 1% reads (backend dashboard).

#### Required SLA

-   **Writes**: Throughput of 100K operations per second.
-   **Reads**: Latency of up to 10 milliseconds for the [99th percentile](https://engineering.linkedin.com/performance/who-moved-my-99th-percentile-latency).

This application requires high availability and fault tolerance. Even if a ScyllaDB node goes down or becomes unavailable, the cluster is expected to
remain available and continue to provide services.

### Creating the Data Model

This section examines the following:
* Data.
* Queries.
* Making the primary key and clustering key selections.
* Creating the database schema.

#### Design and Data Model
The main goal of data modeling in Scylla is to perform fast queries, even if data is sometimes duplicated.

When creating the data model, you need to consider both the conceptual data model and the application workflow: which queries will be performed by which users and how often.

This is achieved by:

* Even data distribution.

* Minimizing the number of partitions accessed in a read query.

![](https://lh5.googleusercontent.com/5JqE89v8KJbSuVsnGswHn83sJOV-tjpeH6r1fqdNl6S77ncqAYb3kIZPSgNI8bqN_43OyZNbHQVpXdqMBFrRmsEvG3JORR302EhMnIb9qa6nuNL7cP2JJDZ4Uon_Pp-QmSCoEQ)

#### Conceptual Data Model
Identify the key entities and the relationships between them.

Our application has pets. Each pet can be tracked by many followers (typically the owners). A follower can also track more than one pet. Each pet can have a few sensors. Each sensor takes measurements:

![](https://lh3.googleusercontent.com/GrFS0HY7XgABabCEp5Fbc0dULsujHkvSykFMiMZRI5TjJTYrzckVCJ29L4BgnEqc8dB7t1_VzsRsJCJjwiNI8xHdQ0tGh9qZptZfRsq7gDXHVogwfJG8Y_DIEJrgLX40zjvV5w)

#### Application Workflow
Identify the main queries to ask the database.

>**Note:**
>This is important in Scylla and other NoSQL databases. By way of contrast to relational databases, this is performed early on in the data modeling process. Remember data modeling is built around the queries.

![](https://lh5.googleusercontent.com/bHN-aBIt-cJ-77s5AWn6Dt0djC-gLQRSArF6b56s3mxpzx-0oG3TgXYOJzTOwrhUUdT0WcEZPTTTdk8B4aY2fbGMgs044DKxwPMC1ATrphI4GBRS8rcVboWpjlg5cO4JRcTZxQ)

#### Queries
The queries above are detailed in [CQL](https://university.scylladb.com/courses/data-modeling/lessons/basic-data-modeling-2/topic/cql-cqlsh-and-basic-cql-syntax/) as follows:

* Query 1: Find a Follower with a specific ID.

  `SELECT * FROM owner WHERE owner_id = ? `

* Query 2: Find the pets that the follower tracks.

  `SELECT * FROM pet WHERE owner_id = ?`  

* Query 3: Find the sensors of a pet.

  `SELECT * FROM sensor WHERE pet_id = ?`  

* Query 4: Find the measurements for a sensor in a date range.

  `SELECT * FROM measurements WHERE sensor_id = ? AND ts <= ? and ts >= ?;`

* Query 5: Find a daily summary of hour based aggregates.

  `SELECT * FROM sensor_avg WHERE sensor_id = ? AND date = ? ORDER BY date ASC, hour ASC;`

#### Logical Data Model
Using the outcomes of the application workflow and the conceptual data model, the logical data model can be created. At this stage, the look of the tables and which fields are used as primary and clustering keys are determined.
>**Note:**
>In Scylla, it is better to duplicate data than to join.

![](https://lh4.googleusercontent.com/zF8v3divX_VsG5z7pOmGOtdLj7_7AVrembG6ep630WsVqJXKMthEoMyPAkfaJsU7a-np9fO84lmfbcHkPv-dX-_45Aczafnm4V7OroHgt0Kd6Ao7vLF6eK_m-d6X5TJcnylpow)

#### Physical Data Model
Add CQL data types to the Logical Data Model above. Make sure you’re familiar with the ScyllaDB (and Cassandra) [data types](https://university.scylladb.com/courses/data-modeling/lessons/advanced-data-modeling/topic/common-data-types-and-collections/) before proceeding.

Based on the high availability requirements, a [replication factor](https://university.scylladb.com/courses/scylla-essentials-overview/lessons/high-availability/topic/fault-tolerance-replication-factor/) (RF) of three is used. The RF is defined when creating the [Keyspace](https://university.scylladb.com/courses/data-modeling/lessons/basic-data-modeling-2/topic/keyspace/). For the tables sensor_avg and measurement, use the [Time Window Compaction Strategy (TWCS)](https://docs.scylladb.com/getting-started/compaction/#time-window-compactionstrategy-twcs).

For the other tables, use the default [compaction strategy, Size Tiered Compaction Strategy (STCS)](https://university.scylladb.com/courses/scylla-operations/lessons/compaction-strategies/topic/size-tiered-and-leveled-compaction-strategies-stcs-lcs/).

>**Note:**
>If you are using [Scylla Enterprise](https://www.scylladb.com/product/scylla-enterprise/), consider using [Incremental Compaction Strategy (ICS)](https://university.scylladb.com/courses/scylla-operations/lessons/compaction-strategies/topic/incremental-compaction-strategy-ics/) as it offers better performance.

The tables are defined according to the physical data model as follows:

```
CREATE TABLE IF NOT EXISTS owner (
    owner_id UUID,
    address TEXT,
    name    TEXT,
    PRIMARY KEY (owner_id)
);

CREATE TABLE IF NOT EXISTS pet (
    owner_id UUID,
    pet_id   UUID,
    chip_id  TEXT,
    species  TEXT,
    breed    TEXT,
    color    TEXT,
    gender   TEXT,
    age     INT,
    weight  FLOAT,
    address TEXT,
    name    TEXT,
    PRIMARY KEY (owner_id, pet_id)
);

CREATE TABLE IF NOT EXISTS sensor (
    pet_id UUID,
    sensor_id UUID,
    type TEXT,
    PRIMARY KEY (pet_id, sensor_id)
);

CREATE TABLE IF NOT EXISTS measurement (
    sensor_id UUID,
    ts       TIMESTAMP,
    value    FLOAT,
    PRIMARY KEY (sensor_id, ts)
) WITH compaction = { 'class' : 'TimeWindowCompactionStrategy' };

CREATE TABLE IF NOT EXISTS sensor_avg (
    sensor_id UUID,
    date    DATE,
    hour    INT,
    value   FLOAT,
    PRIMARY KEY (sensor_id, date, hour)
) WITH compaction = { 'class' : 'TimeWindowCompactionStrategy' };
```

### Deploying the App 

#### Prerequisites

-   [go](https://golang.org/dl/), version 1.14 or newer
-   [docker](https://www.docker.com/)
-   [docker-compose](https://docs.docker.com/compose/)

The example application uses Docker to run a three-node ScyllaDB cluster. It allows tracking of pet's health indicators and consists of three parts:

-  **migrate (/cmd/migrate)**: Creates the CarePet keyspace and tables.
-   **collar (/cmd/sensor)**: Generates a pet health data and pushes it into the
    storage.
-   **web app (/cmd/server)**: REST API service for tracking the pets' health state.

#### Download the code

1. Download the example code from git:

    `$ git clone git@github.com:scylladb/care-pet.git`

2. Create a local ScyllaDB cluster consisting of three nodes:

    `$ docker-compose up -d`

    Docker-compose spins up a ScyllaDB cluster consisting of three nodes; carepet-scylla1, carepet-scylla2 and carepet-scylla3. 

3. Wait for about two minutes and check the status of the cluster:

    `$ docker exec -it carepet-scylla1 nodetool status`

4. Once all the nodes are in Up Normal (UN) status, initialize the database. This creates the keyspaces and tables:

    ```
    $ go build ./cmd/migrate
    $ NODE1=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' carepet-scylla1)
    $ ./migrate --hosts $NODE1
    ```

    Expected output:

    ```
    2020/08/06 16:43:01 Bootstrap database...
    2020/08/06 16:43:13 Keyspace metadata = {Name:carepet DurableWrites:true StrategyClass:org.apache.cassandra.locator.NetworkTopologyStrategy StrategyOptions:map[datacenter1:3] Tables:map[gocqlx_migrate:0xc00016ca80 measurement:0xc00016cbb0 owner:0xc00016cce0 pet:0xc00016ce10 sensor:0xc00016cf40 sensor_avg:0xc00016d070] Functions:map[] Aggregates:map[] Types:map[] Indexes:map[] Views:map[]}
    ```
You can check the database structure with:


```
       $ docker exec -it carepet-scylla1 cqlsh
       
       cqlsh> DESCRIBE KEYSPACES
       carepet  system_schema  system_auth  system  system_distributed  system_traces

       cqlsh> USE carepet;
       cqlsh:carepet> DESCRIBE TABLES
       pet  sensor_avg  gocqlx_migrate  measurement  owner  sensor

       cqlsh:carepet> DESCRIBE TABLE pet

       CREATE TABLE carepet.pet (
           owner_id uuid,
           pet_id uuid,
           address text,
           age int,
           name text,
           weight float,
           PRIMARY KEY (owner_id, pet_id)
       ) WITH CLUSTERING ORDER BY (pet_id ASC)
           AND bloom_filter_fp_chance = 0.01
           AND caching = {'keys': 'ALL', 'rows_per_partition': 'ALL'}
           AND comment = ''
           AND compaction = {'class': 'SizeTieredCompactionStrategy'}
           AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
           AND crc_check_chance = 1.0
           AND dclocal_read_repair_chance = 0.1
           AND default_time_to_live = 0
           AND gc_grace_seconds = 864000
           AND max_index_interval = 2048
           AND memtable_flush_period_in_ms = 0
           AND min_index_interval = 128
           AND read_repair_chance = 0.0
           AND speculative_retry = '99.0PERCENTILE';

       cqlsh:carepet> exit

```

#### Start the pet collar simulation

1. From a separate terminal execute the following command to generate the pet's health data and save it to the database:

    ```
    $ go build ./cmd/sensor
    $ NODE1=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' carepet-scylla1)
    $ ./sensor --hosts $NODE1
    ```

    Expected output:

    ```
    2020/08/06 16:44:33 Welcome to the Pet collar simulator
    2020/08/06 16:44:33 New owner # 9b20764b-f947-45bb-a020-bf6d02cc2224
    2020/08/06 16:44:33 New pet # f3a836c7-ec64-44c3-b66f-0abe9ad2befd
    2020/08/06 16:44:33 sensor # 48212af8-afff-43ea-9240-c0e5458d82c1 type L new measure 51.360596 ts 2020-08-06T16:44:33+02:00
    2020/08/06 16:44:33 sensor # 2ff06ffb-ecad-4c55-be78-0a3d413231d9 type R new measure 36 ts 2020-08-06T16:44:33+02:00
    2020/08/06 16:44:33 sensor # 821588e0-840d-48c6-b9c9-7d1045e0f38c type L new measure 26.380281 ts 2020-08-06T16:44:33+02:00
    ...
    ```

  2. Copy the output to a memorable location.

#### Start the REST API service

Start the REST API service in a separate, third terminal. This server exposes a REST API that allows for tracking the pet's health state:

```
$ go build ./cmd/server
$ NODE1=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' carepet-scylla1)
$ ./server --port 8000 --hosts $NODE1
```

Expected output:

```
2020/08/06 16:45:58 Serving care pet at http://127.0.0.1:8000
```

### Running the Application 

Open http://127.0.0.1:8000/ in a browser or send an HTTP request from the CLI:

```
$ curl -v http://127.0.0.1:8000/
```

Expected output:

```
> GET / HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/7.71.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< Content-Type: application/json
< Date: Thu, 06 Aug 2020 14:47:41 GMT
< Content-Length: 45
< Connection: close
< 
* Closing connection 0
{"code":404,"message":"path / was not found"}
```

The following code 404 in the JSON means everything works as expected.
  `{"code":404,"message":"path / was not found"}`

### REST API Example

In the following examples use the ID from the copy of the pet's health data. The ID is the part after the `#` sign without trailing spaces.

#### Read owner data

```
$ curl -v http://127.0.0.1:8000/api/owner/{owner_id}
```

For example:

```
$ curl http://127.0.0.1:8000/api/owner/a05fd0df-0f97-4eec-a211-cad28a6e5360
```

Expected result:

```
{"address":"home","name":"gmwjgsap","owner_id":"a05fd0df-0f97-4eec-a211-cad28a6e5360"} 
```



### Code Structure and Implementation

The code package structure is as follows:

| Name         | Purpose                             |
| ------------ | ----------------------------------- |
| /api         | swagger api spec                    |
| /cmd         | applications executables            |
| /cmd/migrate | install database schema             |
| /cmd/sensor  | Simulates the pet's collar          |
| /cmd/server  | web application backend             |
| /config      | database configuration              |
| /db          | database handlers (gocql/x)         |
| /db/cql      | database schema                     |
| /handler     | swagger REST API handlers           |
| /model       | application models and ORM metadata |

After data is collected from the pets via the sensors on their collars, it is delivered to the central database for analysis and for health status checking.

The collar code sits in the /cmd/sensor and uses scylladb/gocqlx Go driver to connect to the database directly and publish its data. The collar sends a sensor measurement update once a second.

Overall all applications in this repository use scylladb/gocqlx for:

-   Relational Object Mapping (ORM)
-   Building Queries
-   Migrating database schemas

The web application's REST API server resides in /cmd/server and uses
go-swagger that supports OpenAPI 2.0 to expose its API. API handlers reside in
/handler. Most of the queries are reads.

The application is capable of caching sensor measurements data on an hourly
basis. It uses Lazy Evaluation to manage sensor_avg. It can be viewed as an
application-level lazy-evaluated materialized view. 

The algorithm is simple and resides in /handler/avg.go:

-   Read sensor_avg
-   If there is no data, read measurement data, aggregate in memory, save
-   Serve request


### API examples

#### List the owner's pets

```
$ curl -v http://127.0.0.1:8000/api/owner/{owner_id}/pets
```

For example:

```
$ curl http://127.0.0.1:8000/api/owner/a05fd0df-0f97-4eec-a211-cad28a6e5360/pets
```

Expected result:

```    [{"address":"home","age":57,"name":"tlmodylu","owner_id":"a05fd0df-0f97-4eec-a211-cad28a6e5360","pet_id":"a52adc4e-7cf4-47ca-b561-3ceec9382917","weight":5}]
```

#### List each specific pet's sensor

```
$ curl -v curl -v http://127.0.0.1:8000/api/pet/{pet_id}/sensors
```

For example:

```
$ curl http://127.0.0.1:8000/api/pet/cef72f58-fc78-4cae-92ae-fb3c3eed35c4/sensors
```

Expected result:

```
[{"pet_id":"cef72f58-fc78-4cae-92ae-fb3c3eed35c4","sensor_id":"5a9da084-ea49-4ab1-b2f8-d3e3d9715e7d","type":"L"},{"pet_id":"cef72f58-fc78-4cae-92ae-fb3c3eed35c4","sensor_id":"5c70cd8a-d9a6-416f-afd6-c99f90578d99","type":"R"},{"pet_id":"cef72f58-fc78-4cae-92ae-fb3c3eed35c4","sensor_id":"fbefa67a-ceb1-4dcc-bbf1-c90d71176857","type":"L"}]
```

#### Review the data from a specific sensor

```
$ curl http://127.0.0.1:8000/api/sensor/{sensor_id}/values?from=2006-01-02T15:04:05Z07:00&to=2006-01-02T15:04:05Z07:00
```

For example:

```
$  curl http://127.0.0.1:8000/api/sensor/5a9da084-ea49-4ab1-b2f8-d3e3d9715e7d/values\?from\="2020-08-06T00:00:00Z"\&to\="2020-08-06T23:59:59Z"
```

Expected result:

```
[51.360596,26.737432,77.88015,...]
```

#### Read the pet's daily average per sensor use

```
$ curl http://127.0.0.1:8000/api/sensor/{sensor_id}/values/day/{date}
```

For example:

```
$ curl -v http://127.0.0.1:8000/api/sensor/5a9da084-ea49-4ab1-b2f8-d3e3d9715e7d/values/day/2020-08-06
```

Expected result:

  ```
  [0,0,0,0,0,0,0,0,0,0,0,0,0,0,42.55736]
  ```

### Additional Resources

-   [Scylla Essentials](https://university.scylladb.com/courses/scylla-essentials-overview/) course on Scylla University. It provides an introduction to Scylla and explains the basics.
-   [Data Modeling and Application Development](https://university.scylladb.com/courses/data-modeling/) course on Scylla University. It explains basic and advanced data modeling techniques, including information on workflow application, query analysis, denormalization, and other NoSQL data modeling topics.
-   [Scylla Documentation](https://docs.scylladb.com/).
-   Scylla users [slack channel](http://slack.scylladb.com/).
