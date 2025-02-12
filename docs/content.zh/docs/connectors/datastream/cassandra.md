---
title: Cassandra
weight: 4
type: docs
aliases:
  - /zh/dev/connectors/cassandra.html
  - /zh/apis/streaming/connectors/cassandra.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Apache Cassandra Connector

This connector provides sinks that writes data into a [Apache Cassandra](https://cassandra.apache.org/) database.

<!--
  TODO: Perhaps worth mentioning current DataStax Java Driver version to match Cassandra version on user side.
-->

To use this connector, add the following dependency to your project:

{{< artifact flink-connector-cassandra withScalaVersion >}}

Note that the streaming connectors are currently __NOT__ part of the binary distribution. See how to link with them for cluster execution [here]({{< ref "docs/dev/configuration/overview" >}}).

## Installing Apache Cassandra
There are multiple ways to bring up a Cassandra instance on local machine:

1. Follow the instructions from [Cassandra Getting Started page](http://cassandra.apache.org/doc/latest/getting_started/index.html).
2. Launch a container running Cassandra from [Official Docker Repository](https://hub.docker.com/_/cassandra/)

## Cassandra Source
Flink provides a [FLIP-27](https://cwiki.apache.org/confluence/display/FLINK/FLIP-27%3A+Refactor+Source+Interface)
bounded source to read from Cassandra and return a collection of entities as `DataStream<Entity>`.
An entity is built by Cassandra mapper ([MappingManager](https://javadoc.io/static/com.datastax.cassandra/cassandra-driver-mapping/3.11.2/com/datastax/driver/mapping/MappingManager.html))
based on a POJO containing annotations (as described in Cassandra object mapper).

To use the source, do the following:

```java
ClusterBuilder clusterBuilder = new ClusterBuilder() {    
    @Override    
    protected Cluster buildCluster(Cluster.Builder builder) {      
        return builder.addContactPointsWithPorts(new InetSocketAddress(HOST,PORT))
                      .withQueryOptions(new QueryOptions().setConsistencyLevel(CL))                    
                      .withSocketOptions(new SocketOptions()                   
                      .setConnectTimeoutMillis(CONNECT_TIMEOUT)                    
                      .setReadTimeoutMillis(READ_TIMEOUT))                    
                      .build();    
    }
};  
long maxSplitMemorySize = ... //optional max split size in bytes minimum is 1MB. If not set, maxSplitMemorySize = 64 MB
Source cassandraSource = new CassandraSource(clusterBuilder, 
                                             maxSplitMemorySize, 
                                             Pojo.class, 
                                             "select ... from KEYSPACE.TABLE ...;",
                                             () -> new Mapper.Option[] {Mapper.Option.saveNullFields(true)});    
DataStream<Pojo> stream = env.fromSource(cassandraSource, WatermarkStrategy.noWatermarks(),  "CassandraSource");
```
Regarding performances, the source splits table data like this:
`numSplits = tableSize/maxSplitMemorySize`.

If tableSize cannot be determined or previous numSplits computation makes too few splits, it falls back to `numSplits = parallelism`

## Cassandra Sinks

### Configurations

Flink's Cassandra sink are created by using the static CassandraSink.addSink(DataStream<IN> input) method.
This method returns a CassandraSinkBuilder, which offers methods to further configure the sink, and finally `build()` the sink instance.

The following configuration methods can be used:

1. _setQuery(String query)_
    * Sets the upsert query that is executed for every record the sink receives.
    * The query is internally treated as CQL statement.
    * __DO__ set the upsert query for processing __Tuple__ data type.
    * __DO NOT__ set the query for processing __POJO__ data types.
2. _setClusterBuilder(ClusterBuilder clusterBuilder)_
    * Sets the cluster builder that is used to configure the connection to cassandra with more sophisticated settings such as consistency level, retry policy and etc.
3. _setHost(String host[, int port])_
    * Simple version of setClusterBuilder() with host/port information to connect to Cassandra instances
4. _setMapperOptions(MapperOptions options)_
    * Sets the mapper options that are used to configure the DataStax ObjectMapper.
    * Only applies when processing __POJO__ data types.
5. _setMaxConcurrentRequests(int maxConcurrentRequests, Duration timeout)_
    * Sets the maximum allowed number of concurrent requests with a timeout for acquiring permits to execute.
    * Only applies when __enableWriteAheadLog()__ is not configured.
6. _enableWriteAheadLog([CheckpointCommitter committer])_
    * An __optional__ setting
    * Allows exactly-once processing for non-deterministic algorithms.
7. _setFailureHandler([CassandraFailureHandler failureHandler])_
    * An __optional__ setting
    * Sets the custom failure handler.
8. _setDefaultKeyspace(String keyspace)_
    * Sets the default keyspace to be used.
9. _enableIgnoreNullFields()_
    * Enables ignoring null values, treats null values as unset and avoids writing null fields and creating tombstones.
10. _build()_
    * Finalizes the configuration and constructs the CassandraSink instance.

### Write-ahead Log

A checkpoint committer stores additional information about completed checkpoints
in some resource. This information is used to prevent a full replay of the last
completed checkpoint in case of a failure.
You can use a `CassandraCommitter` to store these in a separate table in cassandra.
Note that this table will NOT be cleaned up by Flink.

Flink can provide exactly-once guarantees if the query is idempotent (meaning it can be applied multiple
times without changing the result) and checkpointing is enabled. In case of a failure the failed
checkpoint will be replayed completely.

Furthermore, for non-deterministic programs the write-ahead log has to be enabled. For such a program
the replayed checkpoint may be completely different than the previous attempt, which may leave the
database in an inconsistent state since part of the first attempt may already be written.
The write-ahead log guarantees that the replayed checkpoint is identical to the first attempt.
Note that that enabling this feature will have an adverse impact on latency.

<p style="border-radius: 5px; padding: 5px" class="bg-danger"><b>Note</b>: The write-ahead log functionality is currently experimental. In many cases it is sufficient to use the connector without enabling it. Please report problems to the development mailing list.</p>

### Checkpointing and Fault Tolerance
With checkpointing enabled, Cassandra Sink guarantees at-least-once delivery of action requests to C* instance.

More details on [checkpoints docs]({{< ref "docs/dev/datastream/fault-tolerance/checkpointing" >}}) and [fault tolerance guarantee docs]({{< ref "docs/connectors/datastream/guarantees" >}})

## Examples

The Cassandra sink currently supports both Tuple and POJO data types, and Flink automatically detects which type of input is used. For general use of those streaming data types, please refer to [Supported Data Types]({{< ref "docs/dev/datastream/fault-tolerance/serialization/types_serialization" >}}#supported-data-types). We show two implementations based on {{< gh_link file="flink-examples/flink-examples-streaming/src/main/java/org/apache/flink/streaming/examples/socket/SocketWindowWordCount.java" name="SocketWindowWordCount" >}}, for POJO and Tuple data types respectively.

In all these examples, we assumed the associated Keyspace `example` and Table `wordcount` have been created.

{{< tabs "ffc5c4d4-7872-479c-bfa6-206b9e96f6f3" >}}
{{< tab "CQL" >}}
```sql
CREATE KEYSPACE IF NOT EXISTS example
    WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'};

CREATE TABLE IF NOT EXISTS example.wordcount (
    word text,
    count bigint,
    PRIMARY KEY(word)
);
```
{{< /tab >}}
{{< /tabs >}}

### Cassandra Sink Example for Streaming Tuple Data Type
While storing the result with Java/Scala Tuple data type to a Cassandra sink, it is required to set a CQL upsert statement (via setQuery('stmt')) to persist each record back to the database. With the upsert query cached as `PreparedStatement`, each Tuple element is converted to parameters of the statement.

For details about `PreparedStatement` and `BoundStatement`, please visit [DataStax Java Driver manual](https://docs.datastax.com/en/developer/java-driver/2.1/manual/statements/prepared/)

{{< tabs "1a84c6a0-0b2f-4f96-8cf8-43ec6dd3bc5d" >}}
{{< tab "Java" >}}
```java
// get the execution environment
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// get input data by connecting to the socket
DataStream<String> text = env.socketTextStream(hostname, port, "\n");

// parse the data, group it, window it, and aggregate the counts
DataStream<Tuple2<String, Long>> result = text
        .flatMap(new FlatMapFunction<String, Tuple2<String, Long>>() {
            @Override
            public void flatMap(String value, Collector<Tuple2<String, Long>> out) {
                // normalize and split the line
                String[] words = value.toLowerCase().split("\\s");

                // emit the pairs
                for (String word : words) {
                    //Do not accept empty word, since word is defined as primary key in C* table
                    if (!word.isEmpty()) {
                        out.collect(new Tuple2<String, Long>(word, 1L));
                    }
                }
            }
        })
        .keyBy(value -> value.f0)
        .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
        .sum(1);

CassandraSink.addSink(result)
        .setQuery("INSERT INTO example.wordcount(word, count) values (?, ?);")
        .setHost("127.0.0.1")
        .build();
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

// get input data by connecting to the socket
val text: DataStream[String] = env.socketTextStream(hostname, port, '\n')

// parse the data, group it, window it, and aggregate the counts
val result: DataStream[(String, Long)] = text
  // split up the lines in pairs (2-tuples) containing: (word,1)
  .flatMap(_.toLowerCase.split("\\s"))
  .filter(_.nonEmpty)
  .map((_, 1L))
  // group by the tuple field "0" and sum up tuple field "1"
  .keyBy(_._1)
  .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
  .sum(1)

CassandraSink.addSink(result)
  .setQuery("INSERT INTO example.wordcount(word, count) values (?, ?);")
  .setHost("127.0.0.1")
  .build()

result.print().setParallelism(1)
```
{{< /tab >}}
{{< /tabs >}}


### Cassandra Sink Example for Streaming POJO Data Type
An example of streaming a POJO data type and store the same POJO entity back to Cassandra. In addition, this POJO implementation needs to follow [DataStax Java Driver Manual](http://docs.datastax.com/en/developer/java-driver/2.1/manual/object_mapper/creating/) to annotate the class as each field of this entity is mapped to an associated column of the designated table using the DataStax Java Driver `com.datastax.driver.mapping.Mapper` class.

The mapping of each table column can be defined through annotations placed on a field declaration in the Pojo class.  For details of the mapping, please refer to CQL documentation on [Definition of Mapped Classes](http://docs.datastax.com/en/developer/java-driver/3.1/manual/object_mapper/creating/) and [CQL Data types](https://docs.datastax.com/en/cql/3.1/cql/cql_reference/cql_data_types_c.html)

{{< tabs "d65ca6f5-acb2-4f2c-b5b6-d986eafca765" >}}
{{< tab "Java" >}}
```java
// get the execution environment
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// get input data by connecting to the socket
DataStream<String> text = env.socketTextStream(hostname, port, "\n");

// parse the data, group it, window it, and aggregate the counts
DataStream<WordCount> result = text
        .flatMap(new FlatMapFunction<String, WordCount>() {
            public void flatMap(String value, Collector<WordCount> out) {
                // normalize and split the line
                String[] words = value.toLowerCase().split("\\s");

                // emit the pairs
                for (String word : words) {
                    if (!word.isEmpty()) {
                        //Do not accept empty word, since word is defined as primary key in C* table
                        out.collect(new WordCount(word, 1L));
                    }
                }
            }
        })
        .keyBy(WordCount::getWord)
        .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))

        .reduce(new ReduceFunction<WordCount>() {
            @Override
            public WordCount reduce(WordCount a, WordCount b) {
                return new WordCount(a.getWord(), a.getCount() + b.getCount());
            }
        });

CassandraSink.addSink(result)
        .setHost("127.0.0.1")
        .setMapperOptions(() -> new Mapper.Option[]{Mapper.Option.saveNullFields(true)})
        .build();


@Table(keyspace = "example", name = "wordcount")
public class WordCount {

    @Column(name = "word")
    private String word = "";

    @Column(name = "count")
    private long count = 0;

    public WordCount() {}

    public WordCount(String word, long count) {
        this.setWord(word);
        this.setCount(count);
    }

    public String getWord() {
        return word;
    }

    public void setWord(String word) {
        this.word = word;
    }

    public long getCount() {
        return count;
    }

    public void setCount(long count) {
        this.count = count;
    }

    @Override
    public String toString() {
        return getWord() + " : " + getCount();
    }
}
```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}
