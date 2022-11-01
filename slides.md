---
layout: cover
theme: seriph
download: true
background: https://user-images.githubusercontent.com/29879298/162908536-c61ef0c1-5bb7-44a2-b971-71031fc24d37.png
class: 'text-center'
highlighter: shiki
lineNumbers: false
info: |
  TiCDC is a tool for replicating the incremental data of TiDB. This document introduces the architecture and key concepts of TiCDC.

  Learn more at [TiFlow](https://github.com/pingcap/tiflow)
drawings:
  persist: false
css: unocss
---

# TiCDC, a tool for replicating the incremental data of TiDB

A Deep Dive

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/pingcap/tiflow" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
Hi, My name is rustin. Today we will talk about TiCDC. It is a tool for replicating the incremental data of TiDB.
-->

---
layout: intro
---
# Rustin Liu


<div class="leading-8 opacity-80">
PingCAPer.<br/>
Data Replication Team.<br/>
TiDB Migration Team Committer.<br/>
TiKV Team Reviewer.<br/>
</div>

<div my-10 grid="~ cols-[40px_1fr] gap-y4" items-center justify-center>
  <div i-ri-github-line op50 ma text-xl/>
  <div><a href="https://github.com/hi-rustin" target="_blank">hi-rustin</a></div>
  <div i-ri-twitter-line op50 ma text-xl/>
  <div><a href="https://twitter.com/hi_rustin" target="_blank">hi_rustin</a></div>
  <div i-ri-user-3-line op50 ma text-xl/>
  <div><a href="https://hi-rustin.rs" target="_blank">hi-rustin.rs</a></div>
</div>

<img src="https://avatars.githubusercontent.com/u/29879298?v=4" rounded-full w-40 abs-tr mt-22 mr-22/>

<div flex="~ gap2">
</div>

<!--
First, let me introduce myself. I am Rustin Liu, a PingCAPer. I am working on the data replication team. I am also a committer of the TiDB migration team. I am also a reviewer of the TiKV team. You can find me on GitHub, Twitter, and my personal website.
-->

---
layout: center
---

<div text-6xl fw100>
  Agenda
</div>

<br>

<div class="grid grid-cols-[3fr_2fr] gap-4">
  <div class="border-l border-gray-400 border-opacity-25 !all:leading-12 !all:list-none my-auto">

  - TiCDC Architecture + Key Metrics
  - Frequently Asked Questions

  </div>
</div>

<style>
h1 {
  font-size: 4rem;
}
</style>

<!--
Today we will talk about TiCDC. We will talk about the architecture of TiCDC and some key metrics. We will also answer some frequently asked questions.
-->

---
layout: intro
---

# What is CDC? & Why we need TiCDC?

<!--
Before we talk about TiCDC. We must first know what CDC is. The full name of CDC is change data capture. In TiDB, there
are some scenarios we need a CDC to capture the data from TiDB.
-->

---

# Data Disaster Recovery

<br/>

<div class="grid grid-cols-2 gap-4 items-center h-100">
  <div class="object-contain h-full of-hidden">
      <span>Data Consistency</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - No requirement</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - Snapshot consistency(sync-point)</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - Eventual consistency(Redo Log)</span>
      <br/>
      <br/>
      <span>Performance Indicators</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - RPO &lt 10S</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - RTO &lt 5min</span>
  </div>
  <div class="object-contain h-full of-hidden w-full">

```plantuml
@startuml

database UpstreamTiKV
database DownstreamTiDB
component S3
package TiCDC {
  component Pull
  component Sort
  component Push
}
Pull -r-> Sort
Sort -r-> Push
Pull -u-> UpstreamTiKV : Fetch
Push --> DownstreamTiDB: Replicate
Sort --> S3: Upload

@enduml
```

  </div>
</div>

<!--
First scenario is data disaster recovery. As you can see, we can fetch data from one TiDB cluster and replicate the data
to another cluster. So some disaster happens, we can just switch our application to use the downstream TiDB
cluster.

And we can store the data in S3 and we can recover the data from S3. There are some requirements for this scenario.
Because we wanna our downstream as an backup. So we need to make sure the data is consistent. There are some levels of
consistency. The first one is no requirement. It means we don't care about the data consistency. The second one is
snapshot consistency. It means we can make sure the data is consistent at a specific point in time. The third one is
eventual consistency. It means we can make sure the data is consistent after a period of time. In TiCDC, we have
a function called sync-point. It can make sure the data is consistent at a specific point in time.

In this scenario, we need to focus on two performance indicators.
The first one is RPO. It means recovery point objective. It means how long we tolerate the data loss. In most cases, we
can guarantee the RPO is less than 10 seconds.

The second one is RTO. It means recovery time objective. It means how long we can recover the cluster to the normal state.
In most cases, we can recover the cluster to the normal state in 5 minutes. In TiCDC, we have a function called
redo log. It can make sure the data is consistent after a period of time.
-->

---

# Data Integration

<br/>

<div class="grid grid-cols-2 gap-4 items-center h-100">
  <div class="object-contain h-full of-hidden">
      <span>Data Format</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - Kafka: Canal-JSON/Avro</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - S3(WIP): CSV</span>
      <br/>
      <br/>
      <br/>
      <span>Performance Indicators</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - Throughput</span>
      <br/>
      <span>&nbsp;&nbsp;&nbsp; - Latency</span>
  </div>
  <div class="object-contain h-full of-hidden w-full">

```plantuml
@startuml

database UpstreamTiKV
component Kafka
component S3
package TiCDC {
  component Pull
  component Sort
  component Push
}
Pull -r-> Sort
Sort -r-> Push
Pull -u-> UpstreamTiKV : Fetch
Push --> Kafka: Produce
Push --> S3: Upload

@enduml
```

  </div>
</div>

<!--
The second scenario is data integration. As you can see, we can fetch data from one TiDB cluster and replicate the data
to another system.

We can convert the changed data to different formats. For example, we can convert the changed data to Kafka. And we can
use Canal-JSON or Avro to encode the data. We can also store the data in S3. And we can use CSV to encode the data.

In this scenario, we also need to focus on two performance indicators. The first one is throughput. It means how many
data we can replicate per second. The second one is latency. It means how big the lag is between the upstream and
the downstream.
-->


---
layout: intro
---

# What is TiCDC?

<!--
Now we know why we need a CDC. Let's talk about TiCDC. How does TiCDC work? Let's take a look at the architecture of
TiCDC.
-->
---

<div class="arch">
<div>

# Architecture

</div>

<div
  class="relation"
>

- A TiCDC cluster has only one owner.
- A capture will have multiple processors.
- A processor can only process one changefeed.
- A changefeed can synchronize multiple tables.

</div>

<div>

```plantuml {scale: 0.8}
@startuml

package "TiKV" {
  gRPC - [TiKV CDC]
}

node "Owner" {
  [Scheduler]
  [DDLSink]
  package "DDL" as DDL1 {
    [OwnerSchemaStorage]
    [OwnerDDLPuller] -u-> [gRPC]
  }
}

node "Processor" {
  [ProcessorMounter]
  [ProcessorSink]
  package "DDL" as DDL2 {
    [ProcessorSchemaStorage]
    [ProcessorDDLPuller] --> [gRPC]
  }
  package "Changefeed1" {
    package "Table1 Pipeline" {
      [Puller1] -u-> [gRPC]
      [Sorter1]
      [TableSink1]
    }
    package "Table2 Pipeline" {
      [Puller2] -u-> [gRPC]
      [Sorter2]
      [TableSink2]
    }
  }
}

database "MySQL/Kafka" {
  [MySQL]
  [Broker]
}

[DDLSink] --> [MySQL]
[DDLSink] --> [Broker]
DDL1 --> [DDLSink]
[OwnerSchemaStorage] ..> [OwnerDDLPuller] : use
[ProcessorSink] --> [MySQL]
[ProcessorSink] --> [Broker]
[Sorter1] ..> [ProcessorMounter] : use
[Sorter2] ..> [ProcessorMounter] : use
[TableSink1] ..> [ProcessorSink] : use
[TableSink2] ..> [ProcessorSink] : use
[ProcessorMounter] ..> DDL2 : use
[Puller1] --> [Sorter1]
[Sorter1] --> [TableSink1]
[Puller2] --> [Sorter2]
[Sorter2] --> [TableSink2]
@enduml
```

</div>
</div>

<style>
.arch {
  display: flex;
}

.arch img {
  margin-top: -80px;
}

.relation {
  position: absolute;
  z-index: 1;
  left: 560px;
  top: 100px;
  font-size: 12px;
  color: black;
}

h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 50%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}
</style>

<!--
As you can see, TiCDC is a TiDB cluster. We call each TiCDC instance a capture. Each capture can have multiple goroutines
to process the data.

We can see there are two types of goroutines. One is the owner. The other one is the processor.
So it is logical concept. You only can have one owner in a TiCDC cluster. But you can have multiple processors.

The owner is responsible for scheduling the tables and executing the DDL.

The processor is responsible for replicating data for a changefeed. Each processor can only process one changefeed.
And a changefeed can replicate multiple tables. So we can see there are multiple pipelines in a processor.

The pipeline is responsible for replicating data for a table. Each pipeline has four components. The first one is the
puller. It is responsible for pulling the data from TiKV. The second one is the sorter. It is responsible for sorting
the data. The third one is the mounter. It is responsible for decoding the data. The last one is the sink. It is responsible
for sending the data to the downstream.

As you can see, the core of TiCDC is the table pipeline. So let's take a look at the table pipeline.
-->

---

# Table Pipeline

Each changefeed creates a processor, and each processor maintains multiple table pipelines.

### Pipeline
<br>
<br>

```mermaid {scale: 1.5}
flowchart LR
    puller((PullerNode)) --> sorter((SorterNode)) --> mounter((MounterNode)) --> sink((SinkNode))
```

<!--
The table pipeline looks like a pipe. The data flows from the puller to the sorter, and then to the mounter, and
finally to the sink. So we can go through the pipeline step by step and see how it replicates data.
-->

---

# Data Flow - An Example

<br/>
<br/>
<br/>

<div class="grid grid-cols-2 gap-4 items-center h-100">
  <div class="object-contain h-full of-hidden">

```sql
-- Create the following table structure--

CREATE TABLE TEST(
   NAME VARCHAR (20)     NOT NULL,
   AGE  INT              NOT NULL,
   PRIMARY KEY (NAME)
);

+-------+-------------+------+------+---------+-------+
| Field | Type        | Null | Key  | Default | Extra |
+-------+-------------+------+------+---------+-------+
| NAME  | varchar(20) | NO   | PRI  | NULL    |       |
| AGE   | int(11)     | NO   |      | NULL    |       |
+-------+-------------+------+------+---------+-------+
```

  </div>
  <div class="object-contain h-full of-hidden w-full">

```sql
-- Execute these two DMs in TiDB --

INSERT INTO TEST (NAME,AGE)
VALUES ('Jack',20);

UPDATE TEST
SET AGE = 25
WHERE NAME = 'Jack';

```

  </div>
</div>

<!--
This is an example. We create a table named TEST. It has two columns. The first one is NAME. It is the primary key.
The second one is AGE. We insert a row into the table. And then we update that row. So we can trace the data flow
in the table pipeline. But before that, let's check the real data structure in TiKV.
-->

---

# Data Flow - Write Data to TiKV

<br/>
<br/>
<br/>

<div class="grid grid-cols-2 gap-4 items-center h-100">
  <div class="object-contain h-full of-hidden">

```sql

INSERT INTO TEST (NAME,AGE)
VALUES ('Jack',20);

+------------+-----------------+
|      Key   |     Value       |
+------------+-----------------+
| TEST_Jack  |    Jack | 20    |
+------------+-----------------+

```

  </div>
  <div class="object-contain h-full of-hidden w-full">

```sql
UPDATE TEST
SET AGE = 25
WHERE NAME = 'Jack';

+------------+-----------------+
|      Key   |     Value       |
+------------+-----------------+
| TEST_Jack  |    Jack | 25    |
+------------+-----------------+

```

  </div>
</div>

<!--
For the insert statement, we can see the key is TEST_Jack. The value is Jack | 20. The key is the table name and the
primary key.

For the update statement, we can see the key is still TEST_Jack. But the value is Jack | 25. So we can see the value is updated.

This just a simplified example. In fact, TiKV uses a more complex key-value structure. But it is not important for
this talk. So we can ignore it.
-->

---

# Data Flow - Puller

Pull DDL and Row Change data from TiKV.

### Pull two regions
```sql {0|all|0}
+--------------------------------------------+-----------------------------------------------------+
|                   Region1                  |                         Region2                     |
+--------------------------------------------+-----------------------------------------------------+
|                                            |              ts3: Test_Mick -> Mick | 18            |
|       ts2: TEST_Jack ->  Jack | 20         |                                                     |
|       ts2: Resolved                        |                                                     |
|       ts4: TEST_Jack ->  Jack | 25         |              ts3: Resolved                          |
|       ts4: Resolved                        |                                                     |
+--------------------------------------------+-----------------------------------------------------+
```


### The real row change data
```sql {0|all|0}
+-------------+--------------+------------+---------+--------------+------------------+------------------+
|   start_ts  |   commit_ts  |  type      | op_type |    key       |       value      |     old_value    |
+-------------+--------------+------------+---------+--------------+------------------+------------------+
|      1      |       2      | COMMITTED  |   PUT   |   TEST_Jack  |     Jack  | 20   |       null       |
|      3      |       4      | COMMITTED  |   PUT   |   TEST_Jack  |     Jack  | 25   |     Jack  | 20   |
+-------------+--------------+------------+---------+--------------+------------------+------------------+
```

<!--
The puller is responsible for pulling the data from TiKV. Let's suppose we have two regions. The first one is
Region1 and it stores the data of the first row. The second one is Region2 and it stores the data of the second
row.

So for our example, data change happened in Region1. Puller pulls the data from Region1.

As you can see, the real row change data has some information. The first one and the second one are the start_ts
and the commit_ts. Because we already know the data is written to TiKV, so the third one is COMMITTED. The fourth
one is the operation type. It is PUT. Because we insert and update the data, so it is PUT.

Then we can see the key and the value. There also has the old_value. It is the old value before the update. Sometimes
we need it.
-->

---

# Data Flow - Sorter

### Before
```sql {0|all|0}
+--------------------------------------+-------------- -----------------------------+
|             Region1                  |                Region2                     |
+--------------------------------------+--------------------------------------------+
|                                      |     ts3: Test_Mick -> Mick | 18            |
|    ts2: TEST_Jack ->  Jack | 20      |                                            |
|    ts2: Resolved                     |                                            |
|    ts4: TEST_Jack ->  Jack | 25      |     ts3: Resolved                          |
|    ts4: Resolved                     |                                            |
+--------------------------------------+--------------------------------------------+
```

### After
```sql {0|all}
+--------------------------------------------+
|                   Events                   |
+--------------------------------------------+
|       ts2: TEST_Jack ->  Jack | 20         |
|       ts2: Resolved                        |
|       ts3: Test_Mick ->  Mick | 18         |
|       ts3: Resolved                        |
|       ts4: TEST_Jack ->  Jack | 25         |
|       ts4: Resolved                        |
+--------------------------------------------+
```

<!--
The sorter is responsible for sorting the data. It is a very important component. Because we receive the data
from different regions. So the data is not in order. We need to sort the data by the commit_ts for one table.

As you can see, we merge the data from Region1 and Region2. And then we sort the data by the commit_ts.

Also, we can see the Resolved event. It is a special event. It means the data before this event is sent to the
TiCDC. No more data will be sent to the TiCDC which has a smaller commit_ts than this event.

This kind of event like a watermark. It is used to advance the progress of the table pipeline. Today I am not gonna
to cover how to calculate the resolved ts. Because it is too complicated.
-->

---

# Data Flow - Mounter

Mounter will use the schema information to convert the row kv
into row changes that TiCDC can handle.

<br/>
<br/>
<br/>

<div grid="~ cols-2 gap-4">
<div>

```go {all|5,7}
type RawKVEntry struct {
	OpType OpType
	Key    []byte
	// nil for delete type
	Value []byte
	// nil for insert type
	OldValue []byte
	StartTs  uint64
	// Commit or resolved TS
	CRTs uint64
	// Additional debug info
	RegionID uint64
}
```

</div>

<div>

```go {all|9,10}
type RowChangedEvent struct {
	StartTs  uint64
	CommitTs uint64
	RowID int64
	Table    *TableName
	ColInfos []rowcodec.ColInfo
	TableInfoVersion uint64
	ReplicaID    uint64
	Columns      []*Column
	PreColumns   []*Column
	IndexColumns [][]int
	ApproximateDataSize int64
}
```

</div>
</div>

<!--
The mounter is responsible for converting the row kv into row changes. Because the row kv is not easy to handle.
We have to use the schema information to convert it.

As you can see, we have Value and OldValue. We can use the schema information to convert them into Columns and
PreColumns.

This kind of row change structure is easy to handle. We can use it to do the downstream data processing. For example,
we can use it to generate the SQL statement or the Kafka message.
-->

---

# Data Flow - Sink

Sink is responsible for sending data to MySQL/TiDB or Kafka.

<div class="sink">

```mermaid {scale: 2}
flowchart LR
    sink((Sink)) --> |database/sql|mysql((MySQL))
    sink((Sink)) --> |Sarama|kafka((Kafka))
    style kafka fill:#f96
```

</div>

<style>
.sink {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>

<!--
The sink is responsible for sending data to the downstream. We can send the data to MySQL/TiDB or Kafka.

We can use the official MySQL driver to send the data to MySQL/TiDB. We can use the Sarama to produce the data to Kafka.

We also have a S3 sink. But it is not ready yet. We will release it in the next version.
-->

---

# Data Flow - Sink

The real SQL or JSON data will be sent to MySQL/TiDB or Kafka.

<br/>

<div class="grid grid-cols-2 gap-4 items-center h-100">
  <div class="object-contain h-full of-hidden">

```sql {0|all|0}
/*
Because there are only Columns,
it is an Insert statement.
*/
INSERT INTO TEST (NAME,AGE)
VALUES ('Jack',20);

/*
Because there are both Columns and PreColumns,
it is an Update statement.
*/
UPDATE TEST
SET AGE = 25
WHERE NAME = 'Jack';

```
  </div>
  <div class="object-contain h-full of-hidden w-full">

```json {0|all}
{
    "id": 0,
    "database": "test",
    "table": "TEST",
    "pkNames": [
        "NAME"
    ],
    "isDdl": false,
    "type": "INSERT",
    ...
    "ts": 2,
    ...
    "data": [
        {
            "NAME": "Jack",
            "AGE": "25"
        }
    ],
    "old": null
}
```

  </div>
</div>

<!--
There are some examples of the real SQL or JSON data. We can see the SQL statement is easy to understand.
We just revert the row change into the SQL statement. You can notice that the SQL statement is not the same
as the original SQL statement.

And the JSON data is also easy to understand. We use the Canal-JSON format. This format developed by Alibaba.
Used by many Chinese companies. We can encode it to a Kafka message and send it to Kafka.

It can be easily decoded by the Flink or other data processing frameworks.
-->


---
layout: intro
---

# Let's check the Data Flow metrics, then you will know the throughput of TiCDC.

<!--
Now you have known the data flow of TiCDC. Let's check the data flow metrics. Then you will know the throughput of TiCDC.
As I said before, the throughput is very important performance indicator.
-->

---
layout: iframe
url: https://metricstool.pingcap.com/viz/index.html#!/
scale: 0.6
---


---
layout: intro
---

# You get the throughput of TiCDC, but how do you know the latency?

<!--
Now you have known the throughput of TiCDC. But how do you know the latency?
There are several metrics to measure the latency. Also, these metrics indicate the latency for different parts of the data flow.
-->

---
layout: iframe
url: https://metricstool.pingcap.com/viz/index.html#!/
scale: 0.6
---

---
layout: intro
---

# Frequently Asked Questions

<!--
Let's move on to the frequently asked questions.
There are many questions about TiCDC. But we can divide them into two categories.
First, people really care about the latency. They always wanna get lower latency.
Second, some people care about the technical details. They wanna know why TiCDC is designed in this way.

So I am gonna cover these two categories. Let's start with the latency.
-->

---
layout: two-cols
---

# Why such a big latency?

<br/>

- Big Transaction
  - Split the big transaction into small ones.(Only v6.1.0+)

- Throughput too low
  - Table memory quota
  - Sink worker count
  - Upstream TiKV region count (Big single table)
  - High workload on Upstream

::right::

<br/>
<br/>
<br/>

- Upstream issue
  - Region Leader Transfer
  - Resolved TS can not advance

- Downstream issue
  - Downstream database is too slow
  - Write conflict

- Cluster topology
  - Cross-Region Deployment

<!--
There are many reasons for the latency. Let's start with the big transaction. If the transaction is too big, it will take a long time to replicate it to the downstream.

So it will cause a big latency. We can split the big transaction into small ones. This feature is only available in v6.1.0+.

The second reason is the throughput is too low. We can increase the throughput by increasing the table memory quota, the sink worker count. As I said before, we replicate the data by table. So we need to control the memory usage of each table. The default
quota is 10MB. You can increase it to get a higher throughput if you have enough memory. You can also increase the sink worker count to get a higher throughput. Because more workers can send the data to the downstream at the same time.

The TiKV region count is also an important factor. If we have too many regions, it will cause TiCDC spends too much time to deal with the Resolved TS. So it will cause a big latency. For now we can not solve this problem. We will improve it in the future.

Also, if your workload is too high on the upstream, it will cause a big latency. Because have no ability to catch up with the upstream.

There are also some upstream issues. For example, the region leader transfer. If the region leader transfer happens, it probably causes a big latency. Because TiCDC needs to re-connect to the new leader. And the Resolved TS can not advance. If the Resolved TS can not advance, it will cause a big latency.

Sometime if your TiDB cluster is crashed, it probably causes a big latency. Because it may leave some locks on the TiKV. And the
resolved TS can not advance. If the locks exist for a long time, it will cause a big latency. But TiCDC will try to resolve the locks if it can.

There are also some downstream issues. For example, if the downstream database is too slow, it will cause a big latency. Because TiCDC can not send the data to the downstream at a high speed. Also, if there are too many write conflicts, it will cause a big latency. Because TiCDC needs to retry the write operation.

The last reason is the cluster topology. If you deploy TiCDC in a cross-region deployment, you should consider the network issue.

That's most of the reasons for the latency. Let's move on to the technical details.
-->

---

# Why this design?

<br/>

- Data Replication Method
  - Raft Learner Vs. Raft Log Event

- Data Order
  - Do we really need to keep the order of the data?

- Write method
  - SQL Vs. Row KV

- Scalability
  - Why scheduler based on table count?

<!--
There are several technical details. Let's start with the data replication method. We can use the Raft Learner to replicate the data. Or we can use the Raft Log Event to replicate the data. We choose the Raft Log Event. Because as I said before, we need to
consider the data integrity and also provide different consistency guarantees. So we can not simply use the Raft Learner.

And I think there are no big performance difference between the two methods. So we choose the Raft Log Event.

About the data order, many people care about the data order and many people don't. So recently we are discussing whether we should keep the data order by default. Maybe we can make it optional in the future.

About the write method, we choose the SQL method. Because it is easy to extend. For example, we can replicate the data to the any MySQL compatible database. So we choose the SQL method instead of the Row KV method. It more flexible.

The last question is about the scalability. Why we schedule the tables based on the table count? Because as you can see, we replicate the data by table. So we simply schedule the tables based on the table count. It is easy to understand and easy to implement.

But sometimes we suffer from the uneven workload. For example, if we have a big table and a small table. The big table will occupy most of the resources. And TiCDC treats them equally. So we are also discussing how to improve it in the future.

Maybe we can schedule the tables based on the workload. But it is not easy to implement. So we are still discussing it.
-->

---
layout: center
---

# Q&A

<br/>
<br/>

## Do you have any questions?

<!--
Thanks for your listening.

Let's move on to the Q&A session. Do you have any questions? I'll try my best to answer them.
-->
