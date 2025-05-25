# Python Cheat Sheet for Data Engineers: Production Reference

*Modern, production-ready guide with prerequisites, error handling, and best practices.*

---

## Prerequisites

* **Python**: version **3.10+**
* **Install dependencies**:

  ```bash
  pip install \
    pandas>=1.5 pyarrow>=11.0 pandera>=0.14 \
    requests>=2.31 sqlalchemy>=1.4 psycopg2-binary>=2.9 \
    boto3>=1.26 confluent-kafka>=1.8 dask[complete]>=2023.2 \
    prefect>=2.0 aiohttp>=3.8 prometheus_client>=0.16 \
    sentry-sdk>=1.19 geopandas>=0.13 redis>=4.5
  ```

  *All versions are minimum for compatibility and reproducibility.*

---

## Environment & Configuration

*Define and manage project settings, secrets & dependencies reliably.*

### Virtual Environments

*Isolate project-specific libraries.*

```bash
# Create project folder and virtualenv
mkdir my_data_project && cd my_data_project
python -m venv venv
# Activate
source venv/bin/activate       # Linux/Mac
.\venv\Scripts\Activate.ps1  # Windows PowerShell
# Verify
which python  # should point to .../venv/bin/python
```

### Dependency Management

*Reproducible installs.*

```bash
# Freeze current packages
pip freeze > requirements.txt
# Install exact versions elsewhere
pip install -r requirements.txt
```

```bash
# Alternatively, use Poetry for lockfile-based management
poetry init --no-interaction && poetry add pandas requests sqlalchemy ...
```

### Environment Variables

*Securely load secrets; never commit `.env` to VCS.*

> **Security Note:** `.env` must be in `.gitignore`. In production, use Vault, AWS Secrets Manager, or similar.

```dotenv
# .env (do not check in)
DB_URL=postgresql://user:pass@host:5432/dbname
MONGO_URI=mongodb://user:pass@host:27017/db
REDIS_URL=redis://localhost:6379/0
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=abcd...
```

```python
# load_env.py
from dotenv import load_dotenv
import os
load_dotenv()  # loads .env
# Access environment
db_url = os.getenv("DB_URL")
mongo_uri = os.getenv("MONGO_URI")
redis_url = os.getenv("REDIS_URL")
```

### Configuration Patterns

*Typed settings with validation.*

```python
# config.py
from pydantic import BaseSettings
class Settings(BaseSettings):
    DB_URL: str            # raises error if missing
    MONGO_URI: str
    REDIS_URL: str
    AWS_REGION: str = "eu-central-1"
    class Config:
        env_file = ".env"
settings = Settings()   # loads & validates
```

---

## Data Ingestion

*Read raw data from files, APIs, or databases with error handling.*

### CSV

```python
# ingest_csv.py
import pandas as pd

def load_csv(path: str) -> pd.DataFrame:
    try:
        df = pd.read_csv(
            path,
            dtype={"id": int},         # pandas>=1.5
            parse_dates=["ts"],       # auto-convert dates
            na_values=[""]            # empty to NaN
        )
        return df
    except Exception as e:
        print(f"Error reading CSV: {e}")
        raise

if __name__ == '__main__':
    df = load_csv("data.csv")
    print(df.head())
```

### JSON

```python
# ingest_json.py
import json
import pandas as pd

def load_and_flatten_json(path: str) -> pd.DataFrame:
    try:
        with open(path) as f:
            records = json.load(f)
        df = pd.json_normalize(
            records,
            record_path=["scores"],
            meta=["id","name"],
            record_prefix="score_"
        )
        return df
    except (json.JSONDecodeError, IOError) as e:
        print(f"Error loading JSON: {e}")
        raise

if __name__ == '__main__':
    print(load_and_flatten_json("data.json").head())
```

### Parquet

```python
# ingest_parquet.py
import pyarrow.parquet as pq
import pandas as pd

def load_parquet(path: str, cols: list[str]) -> pd.DataFrame:
    table = pq.read_table(path, columns=cols)
    return table.to_pandas()
```

### HTTP API

```python
# ingest_api.py
import requests
from requests.exceptions import HTTPError, Timeout

def fetch_items(url: str) -> list[dict]:
    items = []
    params = {"limit":50, "offset":0}
    while True:
        try:
            resp = requests.get(url, params=params, timeout=10)
            resp.raise_for_status()
        except (HTTPError, Timeout) as e:
            print(f"Request error: {e}")
            break
        batch = resp.json().get("results", [])
        if not batch:
            break
        items.extend(batch)
        params["offset"] += params["limit"]
    return items
```

### Database (PostgreSQL)

```python
# ingest_db.py
import os
import pandas as pd
from sqlalchemy import create_engine, text

def pg_query_dataframe(query: str, params: dict | None = None) -> pd.DataFrame:
    try:
        engine = create_engine(os.getenv("DB_URL"))
        with engine.connect() as conn:
            return pd.read_sql(text(query), conn, params=params)
    except Exception as e:
        print(f"DB error: {e}")
        raise
```

---

## Data Transformation

*Clean, normalize, enrich, and reshape data.*

### Pandas

```python
# transform_pandas.py
import pandas as pd
import numpy as np

def transform(df: pd.DataFrame) -> tuple[pd.DataFrame, pd.DataFrame]:
    df = df.copy()
    df["value"] = df["value"].fillna(0)
    df = df[df["value"] > 0]
    df["log_value"] = np.log(df["value"] + 1)
    grouped = df.groupby("category").agg(
        count=("id","count"),
        avg_value=("value","mean")
    )
    return df, grouped
```

### PySpark DataFrame

```python
# transform_spark.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import log, row_number
from pyspark.sql.window import Window

def spark_transform(input_path: str, output_path: str) -> None:
    spark = SparkSession.builder.appName("ETL").getOrCreate()
    df = spark.read.json(input_path)
    clean = (
        df.filter(df.value > 0)
          .withColumn("log_value", log(df.value + 1))
    )
    window = Window.partitionBy("user_id").orderBy(df.timestamp.desc())
    ranked = clean.withColumn("rank", row_number().over(window))
    ranked.write.mode("overwrite").parquet(output_path)
```

### Dask DataFrame

```python
# transform_dask.py
import dask.dataframe as dd

def dask_transform(pattern: str) -> dd.DataFrame:
    ddf = dd.read_csv(pattern)
    result = ddf.groupby("user_id")["value"].sum()
    return result

if __name__=='__main__':
    print(dask_transform("data-*.csv").compute().head())
```

### Validation with Pandera

```python
# validate.py
import pandas as pd
import pandera as pa
from pandera import Column, DataFrameSchema

schema = DataFrameSchema({
    "id": Column(int, nullable=False),
    "value": Column(float, pa.Check(lambda x: x >= 0))
})
df = pd.DataFrame({"id":[1,2],"value":[10,-5]})
schema.validate(df)  # raises on invalid rows
```

---

## Data Storage & Databases

*Write data to SQL, NoSQL, or data warehouses.*

### Relational (SQL)

```python
# store_sql.py
import os
import pandas as pd
from sqlalchemy import create_engine

def bulk_insert(df: pd.DataFrame, table: str) -> None:
    engine = create_engine(os.getenv("DB_URL"))
    df.to_sql(
        table,
        engine,
        if_exists="append",
        index=False,
        method="multi"
    )
```

### MongoDB

```python
# store_mongo.py
import os
import pandas as pd
from pymongo import MongoClient

def insert_documents(df: pd.DataFrame, collection_name: str) -> None:
    client = MongoClient(os.getenv("MONGO_URI"))
    col = client.mydb[collection_name]
    col.insert_many(df.to_dict("records"))
```

### Redis

```python
# store_redis.py
import os
import redis

def cache_set(key: str, value: str) -> None:
    redis_url = os.getenv("REDIS_URL", "redis://localhost:6379/0")
    r = redis.from_url(redis_url, decode_responses=True)
    r.set(key, value)
```

### Data Warehouse (Redshift/Snowflake)

```sql
-- warehouse_load.sql
-- Use IAM roles or external credentials rather than inline keys
COPY my_table
FROM 's3://my-bucket/data/'
IAM_ROLE 'arn:aws:iam::123456789012:role/MyRedshiftRole'
FORMAT AS PARQUET;
```

---

## Geo-Data

*Process spatial data with PostgreSQL/PostGIS and GeoPandas.*

### PostGIS Ingestion

```sql
-- st_create.sql
CREATE TABLE locations (
  id SERIAL PRIMARY KEY,
  name TEXT,
  geom GEOMETRY(Point,4326)
);
COPY locations(name,geom)
FROM 's3://bucket/locations.csv'
WITH CSV;
```

### GeoPandas Example

```python
# transform_geo.py
import geopandas as gpd

def load_and_plot(path: str):
    gdf = gpd.read_file(path)  # supports GeoJSON, Shapefile, etc.
    print(gdf.head())
    gdf.plot()  # requires matplotlib
```

### QGIS Integration

* Load PostGIS layers directly in QGIS via DB connection
* Export GeoPandas to shapefile: `gdf.to_file("out.shp")`

---

## Cloud Storage & Services

*Interact with AWS S3 and GCS.*

### AWS S3

```python
# cloud_s3.py
import boto3
from botocore.exceptions import BotoCoreError, ClientError

def upload_file(local: str, bucket: str, key: str) -> None:
    s3 = boto3.client("s3")
    try:
        s3.upload_file(local, bucket, key)
    except (BotoCoreError, ClientError) as e:
        print(f"S3 upload error: {e}")
```

### Google Cloud Storage

```python
# cloud_gcs.py
from google.cloud import storage

def download_blob(bucket_name: str, blob_name: str, dest: str) -> None:
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    blob.download_to_filename(dest)
```

---

## Messaging & Streaming

*Consume and produce real-time data streams.*

### Apache Kafka

```python
# stream_kafka.py
from confluent_kafka import Producer, Consumer, KafkaError

def produce_message(topic: str, key: str, value: str) -> None:
    p = Producer({"bootstrap.servers":"localhost:9092"})
    p.produce(topic, key=key, value=value)
    p.flush()

def consume_messages(topic: str):
    c = Consumer({
        "bootstrap.servers":"localhost:9092",
        "group.id":"group1",
        "auto.offset.reset":"earliest"
    })
    c.subscribe([topic])
    try:
        while True:
            msg = c.poll(1.0)
            if msg is None:
                continue
            if msg.error():
                print(f"Consumer error: {msg.error()}")
                continue
            print(f"Consumed: {msg.key()}: {msg.value()}")
    except KeyboardInterrupt:
        pass
    finally:
        c.close()
```

### AWS Kinesis

```python
# stream_kinesis.py
import boto3
from botocore.exceptions import BotoCoreError, ClientError

def put_record(stream: str, data: bytes, key: str) -> None:
    client = boto3.client("kinesis")
    try:
        client.put_record(StreamName=stream, Data=data, PartitionKey=key)
    except (BotoCoreError, ClientError) as e:
        print(f"Kinesis error: {e}")
```

---

## Workflow Orchestration

*Coordinate, schedule, and monitor multi-step pipelines.*

### Apache Airflow

> **Note:** Install providers for cloud integrations: `pip install apache-airflow-providers-amazon apache-airflow-providers-google`

```python
# etl_airflow.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime
from my_module import extract, transform, load

with DAG(
    dag_id="etl_pipeline",
    start_date=datetime(2025,5,1),
    schedule_interval="@daily",
    catchup=False
) as dag:
    t1 = PythonOperator(task_id="extract", python_callable=extract)
    t2 = PythonOperator(task_id="transform", python_callable=transform)
    t3 = PythonOperator(task_id="load", python_callable=load)
    t1 >> t2 >> t3
```

### Prefect

```python
# etl_prefect.py
from prefect import flow, task

@task
def extract(): return [1,2,3]

@task
def transform(data): return [d*2 for d in data]

@task
def load(data): print("Loaded", data)

@flow
def etl_flow():
    data = extract()
    data2 = transform(data)
    load(data2)

if __name__ == "__main__":
    etl_flow()
```

---

## Concurrency & Parallelism

*Optimize I/O-bound and CPU-bound tasks.*

### Asyncio

```python
# async_http.py
import asyncio
import aiohttp

async def fetch(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()

async def main():
    urls = ["https://example.com/api1", "https://example.com/api2"]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print(results)

if __name__ == "__main__":
    asyncio.run(main())
```

### Thread & Process Pools

```python
# parallel.py
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def io_task(x: int) -> int:
    return x * 2

def cpu_task(n: int) -> int:
    return sum(i*i for i in range(n))

if __name__ == '__main__':
    with ThreadPoolExecutor(max_workers=5) as tp:
        print(list(tp.map(io_task, range(10))))
    with ProcessPoolExecutor() as pp:
        print(list(pp.map(cpu_task, [1000000]*4)))
```

---

## Testing, Logging & Monitoring

*Ensure correctness, traceability, and system health.*

### Logging

```python
# logging_setup.py
import logging

logging.basicConfig(
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)
logger.info("Pipeline started")
```

### Testing (pytest & coverage)

```python
# test_transform.py
import pandas as pd
import numpy as np
from transform_pandas import transform

def test_transform():
    df = pd.DataFrame({"id":[1],"value":[10],"category":["A"]})
    df_out, grouped = transform(df)
    assert df_out["log_value"][0] == np.log(11)
```

```bash
# run tests with coverage
pytest --maxfail=1 --disable-warnings -q
coverage run -m pytest
coverage report -m
```

### Monitoring & Metrics

```python
# prometheus_export.py
from prometheus_client import start_http_server, Counter
REQUEST_COUNT = Counter('app_requests_total','Total HTTP requests')
start_http_server(8000)
REQUEST_COUNT.inc()
```

```python
# sentry_init.py
import sentry_sdk
sentry_sdk.init(dsn="https://...")
```

---

## Performance & Profiling

*Identify and address bottlenecks.*

### cProfile

```python
# profile_main.py
import cProfile
from my_pipeline import main
cProfile.run("main()", "stats.out")
```

### Line Profiler

```bash
pip install line_profiler
kernprof -l script.py
python -m line_profiler script.py.lprof
```

### Memory Profiler

```bash
pip install memory-profiler
mprof run script.py
mprof plot
```

---

## Best Practices

*Consistent guidelines for production code.*

* **Code Style:** PEP8; run `black .` and `flake8`.
* **Type Hints & Static Analysis:** annotate functions; run `mypy .`.
* **Idempotency:** design tasks to be retry-safe.
* **Logging:** structured logs; include context.
* **Error Handling:** wrap I/O, network, and DB calls in `try/except`.
* **Testing & Coverage:** use pytest and coverage for regression.
* **Profiling:** use cProfile, line\_profiler, memory\_profiler for optimization.
* **Secrets Management:** use managed services (Vault, AWS Secrets Manager).
* **Documentation:** maintain docstrings and a project README.

---

## One-Page Cheat Sheet Summary

| Category         | Command/Pattern                                        |
| ---------------- | ------------------------------------------------------ |
| Virtualenv       | `python -m venv venv` / `source venv/bin/activate`     |
| Install deps     | `pip install -r requirements.txt` / `poetry install`   |
| Load CSV         | `pd.read_csv(path, dtype, parse_dates, na_values)`     |
| HTTP GET         | `resp = requests.get(url); resp.raise_for_status()`    |
| DB Read          | `pd.read_sql(text(query), conn, params)`               |
| Pandas Transform | `df.fillna().query().assign()` / `df.groupby().agg()`  |
| Spark ETL        | `(spark.read).filter().withColumn()...write.parquet()` |
| Dask parallel    | `dd.read_csv().groupby().compute()`                    |
| S3 Upload        | `boto3.client('s3').upload_file(local,bucket,key)`     |
| Kafka Consumer   | `Consumer({...}).poll()` / `msg.value()`               |
| Airflow DAG      | `with DAG(...) as dag: task1>>task2`                   |
| Asyncio          | `asyncio.run(main())` with `aiohttp`                   |
| ThreadPool       | `ThreadPoolExecutor().map()`                           |
| Logging          | `logging.basicConfig(...); logger.info()`              |
| Test             | `pytest` / `coverage run`                              |
| Profile          | `cProfile.run()` / `kernprof` / `mprof run`            |
| GeoPandas        | `gpd.read_file(path).plot()`                           |

*Keep this summary at hand for quick reference.*
