# Big-data-project

# Big Data Engineering Project

## Real‑Time API Data Processing with the Hadoop Ecosystem


![Your paragraph text](https://github.com/user-attachments/assets/d789a20b-5c3b-4c84-850a-236ffe18b11e)

---

### Overview

This project demonstrates a complete big data pipeline designed to collect, ingest, process, and store API data in real time using the Hadoop ecosystem. It was implemented on **CentOS 6.5** and leverages tools such as **Apache Flume, Apache Kafka, Apache Spark**, and **HDFS**. The data originates from the Wikimedia API and is ultimately written to **HDFS** for long-term storage and analysis.

---

### Architecture Summary

| Stage                       | Tool                              | What Happens                                                                                                                                                            |
| --------------------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. **Data Collection**      | Custom Python script              | Consumes the *Server‑Sent Events* (SSE) endpoint `https://stream.wikimedia.org/v2/stream/recentchange` and writes raw JSON files locally.                               |
| 2. **Data Ingestion**       | Apache Flume (SpoolDir → Kafka)   | Monitors the local JSON directory and acts as a Kafka producer, publishing each file to the `wiki_changes` topic.                                                       |
| 3. **Data Streaming**       | Apache Kafka                      | Buffers and distributes the live event stream; provides exactly‑once ordered delivery to downstream consumers.                                                          |
| 4. **Real‑Time Processing** | Apache Spark Structured Streaming | Subscribes to the `wiki_changes` topic, parses the JSON, performs real‑time aggregations (e.g., edits‑per‑minute per wiki), and writes the curated stream back to HDFS. |
| 5. **Raw Storage**          | Apache Flume (Kafka → HDFS)       | A second Flume agent tails Kafka and stores the *verbatim* raw events into HDFS for replay or deep analysis.                                                            |

> **Why both Spark *and* Flume to HDFS?**
> Spark handles low‑latency analytics and enrichment, whereas Flume provides a cheap, fault‑tolerant "data lake" copy of the original payloads. This separation keeps analytical output small while retaining the full event history.

---

### Toolchain

* **OS**: CentOS 6.5 
* **Apache Hadoop 3.3.x** – HDFS storage
* **Apache Kafka 2.8.x** – event streaming
* **Apache Flume 1.9.x** – file→Kafka & Kafka→HDFS bridges
* **Apache Spark 3.3.x** (Structured Streaming)
* **Python 3.9+** with `sseclient` for SSE ingestion

---

### 1 · Python Stream Collector

`connection_stream.py`

```python
# Streams Wikimedia Recent‑Change SSE feed to local JSON files
import json, os
from datetime import datetime
from sseclient import SSEClient   # pip install sseclient-py

OUTPUT_DIR = "/home/bigdata/wiki_logs"
os.makedirs(OUTPUT_DIR, exist_ok=True)

url = "https://stream.wikimedia.org/v2/stream/recentchange"
for event in SSEClient(url):
    if event.event == "message":
        data = json.loads(event.data)
        ts  = datetime.utcnow().strftime("%Y%m%d_%H%M%S%f")
        fn  = os.path.join(OUTPUT_DIR, f"change_{ts}.json")
        with open(fn, "w") as f:
            json.dump(data, f)
        print("Saved", fn)
```

### 2 · Flume Configuration

#### a. File → Kafka (`spool_to_kafka.conf`)

```properties
agent1.sources  = r1
agent1.channels = c1
agent1.sinks    = k1

agent1.sources.r1.type          = spooldir
agent1.sources.r1.spoolDir      = /home/bigdata/wiki_logs
agent1.sources.r1.channels      = c1

agent1.sinks.k1.type            = org.apache.flume.sink.kafka.KafkaSink
agent1.sinks.k1.kafka.topic     = wiki_changes
agent1.sinks.k1.kafka.bootstrap.servers = localhost:9092
agent1.sinks.k1.channel         = c1

agent1.channels.c1.type = memory
```

#### b. Kafka → HDFS (`kafka_to_hdfs.conf`)

```properties
agent2.sources  = r2
agent2.channels = c2
agent2.sinks    = h2

agent2.sources.r2.type              = org.apache.flume.source.kafka.KafkaSource
agent2.sources.r2.kafka.bootstrap.servers = localhost:9092
agent2.sources.r2.kafka.topics      = wiki_changes
agent2.sources.r2.channels          = c2

agent2.sinks.h2.type                = hdfs
agent2.sinks.h2.hdfs.path           = hdfs://localhost:9000/user/bigdata/flume/raw/wiki_changes
agent2.sinks.h2.channel             = c2

agent2.channels.c2.type = memory
```
---

### 3 · Kafka Topic

```bash
bin/kafka-topics.sh --create \
  --topic wiki_changes \
  --bootstrap-server localhost:9092 \
  --partitions 1 --replication-factor 1
```

---

### 4 · Spark Structured Streaming Job 

`data_streaming.py`

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col, window
from pyspark.sql.types import *

spark = (SparkSession.builder
         .appName("WikiChangesStreaming")
         .getOrCreate())

schema = StructType().add("$schema", StringType()).add("meta", StructType())\
                       .add("wiki", StringType()).add("type", StringType())\
                       .add("title", StringType()).add("timestamp", LongType())

raw = (spark.readStream.format("kafka")
        .option("kafka.bootstrap.servers", "localhost:9092")
        .option("subscribe", "wiki_changes")
        .load())

json_df = raw.select(from_json(col("value").cast("string"), schema).alias("data"))

edits_per_min = (json_df
                 .select(col("data.wiki").alias("wiki"),
                         (col("data.timestamp").cast("timestamp")).alias("ts"))
                 .groupBy(window(col("ts"), "1 minute"), col("wiki"))
                 .count())

query = (edits_per_min.writeStream
         .outputMode("complete")
         .format("parquet")
         .option("path", "/user/bigdata/aggregates/wiki_edits_minute")
         .option("checkpointLocation", "/user/bigdata/checkpoints/wiki_edits_minute")
         .start())

query.awaitTermination()
```

*Produces a continuously updated **Parquet table** in HDFS containing *edits‑per‑minute per wiki*.*

---

---

### 5 · Running Everything

```bash
# 1.  Start Hadoop  HDFS and YARN
start-dfs.sh  
start-yarn.sh

# 2.  Start ZooKeeper & Kafka
$KAFKA_HOME/bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
$KAFKA_HOME/bin/kafka-server-start.sh    -daemon config/server.properties

# 3.  Create the topic
$KAFKA_HOME/bin/kafka-topics.sh --create --topic wiki_changes \
                                --bootstrap-server localhost:9092 \
                                --partitions 1 --replication-factor 1

# 4.  Launch Flume agents
$FLUME_HOME/bin/flume-ng agent --conf conf --conf-file spool_to_kafka.conf \
                               --name agent1 -Dflume.root.logger=INFO,console &
$FLUME_HOME/bin/flume-ng agent --conf conf --conf-file kafka_to_hdfs.conf \
                               --name agent2 -Dflume.root.logger=INFO,console &

# 5.  Start the Python collector
python3 connection_stream.py &

# 6.  Submit the Spark job 
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.1 \
             data_streaming.py
```

---

### 6 · Outputs

* **Raw events** → `hdfs:///user/bigdata/flume/raw/wiki_changes/*`
* **Aggregated edits‑per‑minute** → `hdfs:///user/bigdata/aggregates/wiki_edits_minute/`
```

```
### 7 · Repository Structure 

```
├── connection_stream.py
├── data_streaming.py
├── flume
│   ├── spool_to_kafka.conf
│   └── kafka_to_hdfs.conf
├── gifs
│   └── Big data tools diagram.gif
└── README.md ← this document
```



## Next Steps

* Monitor throughput & latency using Kafka JMX metrics and Spark UI.
* Replace Flume with **Kafka Connect + Sinks** for a production‑grade pipeline.
* Add Airflow/Dagster to orchestrate and retry each stage.
