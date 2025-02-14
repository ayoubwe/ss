# Real-Time Tweet Processing with Apache Kafka, Spark & Hive on Windows

## Overview

This project sets up a **real-time tweet processing pipeline** using **Apache Kafka, Apache Spark, and Apache Hive** on Windows. The pipeline streams tweets, processes them using Spark, and stores the results in Hive.

## Features

- **Kafka Producer** to send simulated tweets.
- **Spark Consumer** to process tweets and extract:
  - Total number of words.
  - Number of unique words.
  - Number of short words (<= 4 letters).
  - Number of long words (> 6 letters).
- **Hive Storage** to store processed tweet data.

## Prerequisites

- [Java JDK 8+](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)
- [Apache Kafka](https://kafka.apache.org/downloads)
- [Apache Spark](https://spark.apache.org/downloads.html)
- [Apache Hive](https://hive.apache.org/downloads.html)
- [Python 3](https://www.python.org/downloads/)
- [Hadoop Winutils](https://github.com/steveloughran/winutils)
- Python libraries:
  ```sh
  pip install kafka-python pyspark
  ```

## Setup Instructions

### 1. Install & Configure Kafka

1. Extract Kafka and navigate to its directory.
2. Start **Zookeeper**:
   ```sh
   .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
   ```
3. Start **Kafka**:
   ```sh
   .\bin\windows\kafka-server-start.bat .\config\server.properties
   ```
4. Create a Kafka topic:
   ```sh
   .\bin\windows\kafka-topics.bat --create --topic tweets --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
   ```

### 2. Configure Spark

1. Extract Spark and set `SPARK_HOME` in environment variables.
2. Verify installation:
   ```sh
   pyspark
   ```

### 3. Configure Hive

1. Extract Hive and set `HIVE_HOME` in environment variables.
2. Initialize Hive schema:
   ```sh
   schematool -dbType derby -initSchema
   ```
3. Start Hive CLI:
   ```sh
   hive
   ```

## Running the Project

### 1. Kafka Producer (`producer.py`)

```python
from kafka import KafkaProducer
import json, time

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

tweets = [
    "Hello world! This is a test tweet.",
    "Apache Kafka and Spark make real-time processing easy!",
    "Big data analytics is powerful in modern applications.",
    "Short words like a, the, and is are common.",
    "Real-time systems are amazing."
]

for tweet in tweets:
    producer.send('tweets', {'text': tweet})
    print(f"Sent: {tweet}")
    time.sleep(1)
```

**Run:**

```sh
python producer.py
```

### 2. Spark Consumer (`transformer.py`)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, size, split, expr

spark = SparkSession.builder \
    .appName("TwitterStreamProcessor") \
    .config("spark.sql.catalogImplementation", "hive") \
    .enableHiveSupport() \
    .getOrCreate()

tweets_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "tweets") \
    .option("startingOffsets", "earliest") \
    .load()

tweets_df = tweets_df.selectExpr("CAST(value AS STRING) as text")

tweets_df = tweets_df.withColumn("words", split(col("text"), " "))

tweets_df = tweets_df.withColumn("word_count", size(col("words"))) \
    .withColumn("unique_word_count", expr("size(array_distinct(words))")) \
    .withColumn("short_word_count", expr("size(filter(words, w -> length(w) <= 4))")) \
    .withColumn("long_word_count", expr("size(filter(words, w -> length(w) > 6))"))

query = tweets_df.writeStream \
    .format("hive") \
    .option("checkpointLocation", "/tmp/checkpoints") \
    .option("path", "/user/hive/warehouse/tweets_data") \
    .outputMode("append") \
    .start()

query.awaitTermination()
```

**Run:**

```sh
python transformer.py
```

### 3. Create Hive Table

```sql
CREATE TABLE tweets_data (
    text STRING,
    word_count INT,
    unique_word_count INT,
    short_word_count INT,
    long_word_count INT
)
STORED AS PARQUET;
```

### 4. Query Results

```sql
SELECT * FROM tweets_data LIMIT 10;
```

## Summary

1. **Start Kafka & Zookeeper**:
   ```sh
   .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
   .\bin\windows\kafka-server-start.bat .\config\server.properties
   ```
2. **Run Producer**:
   ```sh
   python producer.py
   ```
3. **Run Spark Consumer**:
   ```sh
   python transformer.py
   ```
4. **Check Hive Table**:
   ```sql
   SELECT * FROM tweets_data LIMIT 10;
   ```

### ✅ The pipeline is now running and processing tweets in real time! 🚀



