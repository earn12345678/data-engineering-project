# End-to-End Data Engineering Pipeline

A real-world data pipeline that streams product recall data from a public API through **Apache Kafka**, processes it with **Apache Spark**, stores it in **PostgreSQL**, and orchestrates everything with **Apache Airflow** — all containerized with **Docker**.

---

## Architecture

![Architecture Diagram](pipeline_diagram.svg)

| Component | Technology | Role |
|-----------|------------|------|
| Data Source | RappelConso API (French Gov) | ~10,000 product recall records |
| Message Broker | Apache Kafka + Kafka UI | Stream ingestion & buffering |
| Processing | Apache Spark (PySpark) | Transform & load data |
| Storage | PostgreSQL + pgAdmin 4 | Persistent data storage |
| Orchestration | Apache Airflow | Schedule & monitor DAG tasks |
| Infrastructure | Docker + Docker Compose | Containerized services |

---

## How It Works

The pipeline runs as a 2-task Airflow DAG (`kafka_spark_dag`) scheduled daily:

```
RappelConso API
      │
      ▼  (PythonOperator)
   Kafka Topic: rappel_conso
      │
      ▼  (DockerOperator)
   PySpark Structured Streaming
      │
      ▼
   PostgreSQL: rappel_conso_table
```

**Task 1 — `kafka_data_stream`**: A Python script fetches product recall records from the API and produces messages to a Kafka topic. It tracks the last processed date in `data/last_processed.json` to avoid re-ingesting old records on subsequent runs.

**Task 2 — `pyspark_consumer`**: A PySpark job consumes messages from the Kafka topic, applies a schema transformation, deduplicates against existing records, and writes new data to PostgreSQL via JDBC.

---

## Tech Stack

- **Apache Kafka** `bitnami/kafka` — distributed message streaming
- **Apache Spark** `bitnami/spark` — structured streaming with PySpark
- **Apache Airflow** — workflow orchestration with LocalExecutor
- **PostgreSQL** — relational data storage
- **Docker & Docker Compose** — containerized infrastructure
- **Python** — Kafka producer, data transformation scripts

---

## Prerequisites

- Docker Desktop (8 GB RAM minimum recommended)
- Python 3.8+
- PostgreSQL + pgAdmin 4
- Git

---

## Setup & Run

### 1. Clone the repository

```bash
git clone https://github.com/YOUR-USERNAME/data-engineering-project.git
cd data-engineering-project
```

### 2. Install Python dependencies

```bash
pip install -r requirements.txt
```

### 3. Set up PostgreSQL

Install PostgreSQL from [postgresql.org](https://www.postgresql.org/download/) and create a database, then run:

```bash
export POSTGRES_PASSWORD="your_password"
python scripts/create_table.py
```

### 4. Create Docker network

```bash
docker network create airflow-kafka
```

### 5. Start Kafka

```bash
docker-compose up -d
```

Open Kafka UI at **http://localhost:8800** and create a topic named `rappel_conso` (1 partition, 1 replication factor, time to retain: 1 hour).

### 6. Build Spark Docker image

```bash
docker build -f spark/Dockerfile \
  -t rappel-conso/spark:latest \
  --build-arg POSTGRES_PASSWORD=$POSTGRES_PASSWORD .
```

### 7. Create Airflow environment variables

```bash
echo -e "AIRFLOW_UID=$(id -u)\nAIRFLOW_PROJ_DIR=\"./airflow\"" > .env
```

### 8. Start Airflow

```bash
docker compose -f docker-compose-airflow.yaml up -d
```

Open Airflow UI at **http://localhost:8080** (username: `airflow`, password: `airflow`).

### 9. Trigger the pipeline

In the Airflow UI, enable the `kafka_spark_dag` and click **Trigger DAG**. The pipeline will run both tasks sequentially and load ~10,000 records into PostgreSQL.

---

## 🔍 Verify Results

Open pgAdmin 4 and query the database:

```sql
SELECT COUNT(*) FROM rappel_conso_table;

SELECT categorie_de_produit, COUNT(*) AS total
FROM rappel_conso_table
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;
```

---

## 📁 Project Structure

```
├── airflow/
│   ├── Dockerfile
│   ├── __init__.py
│   └── dags/
│       ├── __init__.py
│       └── dag_kafka_spark.py      # Airflow DAG definition
├── data/
│   └── last_processed.json         # Tracks last ingestion date
├── kafka/
├── scripts/
│   └── create_table.py             # Creates PostgreSQL table
├── spark/
│   └── Dockerfile
├── src/
│   ├── __init__.py
│   ├── constants.py
│   ├── kafka_client/
│   │   ├── __init__.py
│   │   └── kafka_stream_data.py    # Kafka producer
│   └── spark_pgsql/
│       └── spark_streaming.py      # PySpark consumer
├── docker-compose-airflow.yaml     # Airflow services
├── docker-compose.yml              # Kafka services
└── requirements.txt
```

---

## 📚 Credits

Based on the original project by [Hamza Gharbi](https://github.com/HamzaG737/data-engineering-project).
