# Crypto Analysis Data Pipeline

A real-time cryptocurrency data pipeline that fetches market data from CoinGecko API, processes it using Apache Kafka and Apache Spark, and performs analytics with results stored in PostgreSQL.

## 🏗️ Architecture

```
CoinGecko API → Kafka Producer → Kafka Topic → Spark Streaming → Parquet Files → Analytics → PostgreSQL
```

## 📋 Features

- **Real-time Data Ingestion**: Fetches cryptocurrency prices from CoinGecko API every 30 seconds
- **Stream Processing**: Uses Apache Kafka for message queuing and Apache Spark for stream processing
- **Data Storage**: Stores raw data in Parquet format for efficient querying
- **Analytics**: Calculates technical indicators including:
  - 1-minute and 5-minute price changes
  - Simple Moving Average (SMA)
  - Exponential Moving Average (EMA)
  - Volatility metrics
  - Top 5 gainers and losers
- **Database Integration**: Stores processed metrics in PostgreSQL

## 🛠️ Prerequisites

- **Java JDK 11+** (required for Kafka and Spark)
- **Python 3.7+**
- **Apache Kafka 3.7.0** (Scala 2.12 compatible)
- **Apache Spark 3.5.5**
- **PostgreSQL 17+**
- **PostgreSQL JDBC Driver** (`postgresql-42.7.5.jar`)


1. Create database:
```sql
CREATE DATABASE crypto_metrics;
\c crypto_metrics
```

2. Create tables:
```sql
-- Main metrics table
CREATE TABLE crypto_table (
    timestamp   TIMESTAMP,
    id          TEXT,
    symbol      TEXT,
    price       DOUBLE PRECISION,
    change_1min DOUBLE PRECISION,
    change_5min DOUBLE PRECISION,
    sma         DOUBLE PRECISION,
    ema         DOUBLE PRECISION,
    volatility  DOUBLE PRECISION
);

-- Top 5 gainers table
CREATE TABLE top_5_gainers (
    rank   INTEGER NOT NULL,
    id     TEXT,
    symbol TEXT
);

-- Top 5 losers table
CREATE TABLE top_5_losers (
    rank   INTEGER NOT NULL,
    id     TEXT,
    symbol TEXT
);
```

## 🚀 Running the Pipeline

### Step 1: Start Zookeeper

Open a terminal and navigate to Kafka directory:
```bash
cd C:\Kafka
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
```

### Step 2: Start Kafka Server

Open a **new** terminal:
```bash
cd C:\Kafka
bin\windows\kafka-server-start.bat config\server.properties
```

### Step 3: Create Kafka Topic

Open a **new** terminal:
```bash
cd C:\Kafka
bin\windows\kafka-topics.bat --create --topic crypto-prices --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

To delete and recreate the topic (if needed):
```bash
bin\windows\kafka-topics.bat --delete --topic crypto-prices --bootstrap-server localhost:9092
bin\windows\kafka-topics.bat --create --topic crypto-prices --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

### Step 4: Start Data Producer

Open a **new** terminal (with virtual environment activated):
```bash
cd <project_directory>
python dags/crypto_dag.py
```

This will start fetching data from CoinGecko API and publishing to Kafka every 30 seconds.

### Step 5: Start Spark Streaming Consumer

Open a **new** terminal:
```bash
cd <project_directory>/spark_streaming
```

**Important**: Update the paths in `kafka_streaming.py`:
- Line 61: `output_path = "file:///Path to output folder"`
- Line 62: `checkpoint_path = "file:///Path to checkpoint folder"`

Then run:
```bash
spark-submit --repositories https://repo.maven.apache.org/maven2 --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.5 --master local[*] kafka_streaming.py
```

This will consume messages from Kafka and write to Parquet files.

### Step 6: Start Analytics Job

Open a **new** terminal:
```bash
cd <project_directory>/spark_streaming
```

**Important**: Update the path in `analytics.py`:
- Line 32: `"file:///Path to ouput folder"` (should match the output_path from Step 5)

Also update PostgreSQL credentials in `analytics.py` (lines 16-24):
```python
jdbc_url = "jdbc:postgresql://localhost:5432/crypto_metrics"
db_properties = {
    "user": "postgres",
    "password": "your_password",  # Update this
    "driver": "org.postgresql.Driver"
}
```

Then run:
```bash
spark-submit --jars postgresql-42.7.5.jar analytics.py
```

This will process Parquet files, calculate metrics, and write to PostgreSQL every 6 minutes.

## 📊 Monitoring

### View Kafka Messages

To verify data is flowing through Kafka:
```bash
cd C:\Kafka
bin\windows\kafka-console-consumer.bat --topic crypto-prices --bootstrap-server localhost:9092 --from-beginning
```

### Query PostgreSQL

Connect to the database:
```bash
psql -h localhost -p 5432 -U postgres -d crypto_metrics
```

View data:
```sql
-- View latest metrics
SELECT * FROM crypto_table ORDER BY timestamp DESC LIMIT 20;

-- View top gainers
SELECT * FROM top_5_gainers;

-- View top losers
SELECT * FROM top_5_losers;
```

