# setup jupypter+spark container

The Jupyter Docker project already provides a pyspark-notebook image containing Jupyter, Python, Apache Spark, Hadoop binaries, PyArrow, etc.

## Simplest: one container

You can actually get started without Docker Compose.

### 1. Create a directory

```text
pyspark/
├── notebooks/
└── docker-compose.yml
```

### 2. add to `docker-compose.yml`

```dockerfile
services:
  jupyter:
    image: quay.io/jupyter/pyspark-notebook:latest
    container_name: pyspark-jupyter
    ports:
      - "8888:8888"
      - "4040:4040"
    volumes:
      - ./notebooks:/home/jovyan/work
```

### 3. run

```bash
docker compose up
```

You'll see something like:

```text
http://127.0.0.1:8888/lab?token=xxxxxxxx
```

### 4. Open that URL in your browser

That's it.

You now have:

```text
        Your laptop
            │
            │ browser
            ▼
        localhost:8888
            │
            ▼
┌──────────────────────────────┐
│ Docker Container             │
│                              │
│ JupyterLab                   │
│ Python                       │
│ PySpark                      │
│ Apache Spark                 │
│ Java                         │
│ Hadoop libraries             │
│                              │
│ /home/jovyan/work            │
│           ▲                  │
└───────────┼──────────────────┘
            │
            │ mounted volume
            ▼
       ./notebooks/
```

_Note: The Jupyter Docker documentation specifically provides pyspark-notebook for this use case._

### 5. Test it

Create a notebook in Jupyter and run:

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("MyFirstSparkApp")
    .master("local[*]")
    .getOrCreate()
)

df = spark.createDataFrame(
    [
        (1, "Alice", 25),
        (2, "Bob", 30),
        (3, "Charlie", 35),
    ],
    ["id", "name", "age"]
)

df.show()
```

You should get:

```text
+---+-------+---+
| id|   name|age|
+---+-------+---+
|  1|  Alice| 25|
|  2|    Bob| 30|
|  3|Charlie| 35|
+---+-------+---+
```

You can also access Spark's web UI at:

```text
http://localhost:4040
```

_Note: The Jupyter Spark image exposes Spark UI on port 4040._

### 6. What gets installed on your computer?

Very little.

You need:

```text
Docker
```

That's basically it.

You **don't** need:

```text
❌ Python
❌ pip
❌ Java
❌ PySpark
❌ Apache Spark
❌ Hadoop
❌ Jupyter
❌ JupyterLab
```

Those live inside the container.

#### Your workflow becomes

```bash
docker compose up
```

→ open Jupyter in browser

→ write PySpark code

→ stop with:

```bash
docker compose down
```

#### Your notebooks remain on your computer

This part is important:

```dockerfile
volumes:
  - ./notebooks:/home/jovyan/work
```

means:

```text
./notebooks
```

on your computer is mounted into:

```text
/home/jovyan/work
```

inside the container.

So if you create:

```text
notebooks/
    pyspark-basics.ipynb
    dataframe.ipynb
    sql.ipynb
```

those files remain on your machine even after you destroy the container.

This is the recommended pattern for persisting notebooks/data with Docker Stacks.

### But there's an important distinction

There are two different things you might mean by "PySpark with Docker."

#### A. Spark running locally inside the container

```text
Jupyter
   │
   ▼
PySpark
   │
   ▼
Spark
   │
   └── local[*]
```

This is what I recommend **initially**.

You get a real Spark environment, but Spark runs in local mode inside the Jupyter container.

For learning DataFrames, SQL, transformations, joins, aggregations, window functions, MLlib, etc., this is excellent.

The Jupyter documentation explicitly describes local mode as useful for experimentation when you don't have a Spark cluster.

#### B. Actual Spark cluster

Later you could have:

```text
                    ┌───────────────┐
                    │   Jupyter     │
                    │   Notebook    │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Spark Master  │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ Worker 1 │  │ Worker 2 │  │ Worker 3 │
        └──────────┘  └──────────┘  └──────────┘
```

That's also possible with Docker Compose.

But **don't start there unless you specifically want to learn Spark cluster architecture**.

## A better project structure

Once you're comfortable with the basic setup, I'd make it slightly more professional:

```text
pyspark-project/
│
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
│
├── notebooks/
│ ├── 01-introduction.ipynb
│ ├── 02-dataframes.ipynb
│ └── 03-spark-sql.ipynb
│
├── data/
│ ├── input/
│ └── output/
│
└── src/
└── jobs/
```

Then you can mount both notebooks and data:

```dockerfile
services:
  jupyter:
    image: quay.io/jupyter/pyspark-notebook:latest
    container_name: pyspark-jupyter

    ports:
      - "8888:8888"
      - "4040:4040"

    volumes:
      - ./notebooks:/home/jovyan/work
      - ./data:/home/jovyan/data
      - ./src:/home/jovyan/src
```

Now your container sees:

```text
/home/jovyan/
│
├── work/ ← notebooks
├── data/ ← datasets
└── src/ ← Python code
```

while your computer sees:

```text
pyspark-project/
│
├── notebooks/
├── data/
└── src/
```

### What about installing Python packages?

This is where I'd recommend not doing this:

```bash
docker exec -it pyspark-jupyter bash
pip install pandas
```

because that installation belongs to the current container and isn't the best way to make your environment reproducible.

Instead, eventually create your own Docker image:

```dockerfile
FROM quay.io/jupyter/pyspark-notebook:latest

RUN pip install --no-cache-dir \
 requests \
 openpyxl \
 sqlalchemy
```

Then:

```dockerfile
services:
  jupyter:
    build: .
    ports:
      - "8888:8888"
      - "4040:4040"
    volumes:
      - ./notebooks:/home/jovyan/work
      - ./data:/home/jovyan/data
```

And:

```bash
docker compose build
docker compose up
```

That gives you a reproducible development environment.

The Jupyter Docker Stacks documentation recommends building a child image when you want additional packages to persist across container recreation.

_Note: Use a specific image tag instead of latest, so your environment doesn't unexpectedly change later._
