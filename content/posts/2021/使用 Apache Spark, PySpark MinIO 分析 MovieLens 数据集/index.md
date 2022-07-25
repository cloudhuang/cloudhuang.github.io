---
title: 使用 Apache Spark, PySpark MinIO 分析 MovieLens 数据集
date: 2021-10-29
lastmod: 2022-07-25
tags: [publish, 大数据, Spark, PySpark]
---

使用 Apache Spark/PySpark MinIO 分析 MovieLens 数据集

## MinIO
MinIO是一个用Golang开发的基于GNU Affero General Public License v3.0开源协议的高性能对象存储服务。

MinIO兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等。它可以部署在企业的内部环境中，为机器学习、分析和应用数据工作负载建立高性能的存储基础设施。

MinIO用作云原生应用程序的主要存储解决方案，可以与Kubernetes相结合，成为Hadoop生态系统中存储部分的一个有趣的替代品。

## Apache Spark / PySpark
Apache Spark 是一种用于大数据工作负载的分布式开源处理系统。它使用内存中缓存和优化的查询执行方式，可针对任何规模的数据进行快速分析查询。它提供使用 Java、Scala、Python 和 R 语言的开发 API，支持跨多个工作负载重用代码—批处理、交互式查询、实时分析、机器学习和图形处理等。

Apache Spark是用Scala编程语言编写的。PySpark的发布是为了支持Apache Spark和Python的协作，它实际上是Spark的一个Python API。此外，PySpark帮助你在Apache Spark和Python编程语言中与弹性分布式数据集（RDDs）对接。这是通过利用Py4J库实现的。Py4J是一个流行的库，它集成在PySpark中，允许python动态地与JVM对象对接。

![PySpark-1024x164.png](https://ae05.alicdn.com/kf/Hbd09a67a761b449fa201263911aec493y.png)

## Jupyter Notebook
Jupyter Notebook是一个开源的WEB应用，允许用户创建和分享包含实时代码、方程式、可视化和叙述性文本的文档。其用途包括数据分析、统计建模、数据可视化、机器学习等等。Jupyter这个词是Julia、Python和R的松散缩写，不过现在Jupyter已经可以支持许多其他的编程语言。

## MovieLens 数据集
GroupLens Research从MovieLens网站 (https://movielens.org) 收集并提供了电影评分数据集，总共58098个电影，包含了27753444个评价和1108997个标签。

数据集下载地址： http://files.grouplens.org/datasets/movielens/ml-latest.zip


下面就主要使用这几个技术来对数据集进行简单的分析，并最终将分析结果保存的数据库中。

## 准备环境
这里使用 docker compose 部署 4 个节点的 MinIO 集群，然后通过 Nginx 进行反向代理

```
version: '3.7'

# Settings and configurations that are common for all containers
x-minio-common: &minio-common
  image: minio/minio
  command: server --console-address ":9001" http://minio{1...4}/data{1...2}
  expose:
    - "9000"
    - "9001"
  environment:
    MINIO_ROOT_USER: minio
    MINIO_ROOT_PASSWORD: minio123
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
    interval: 30s
    timeout: 20s
    retries: 3

# starts 4 docker containers running minio server instances.
# using nginx reverse proxy, load balancing, you can access
# it through port 9000.
services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - ./minio/data/data1-1:/data1
      - ./minio/data/data1-2:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - ./minio/data/data2-1:/data1
      - ./minio/data/data2-2:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - ./minio/data/data3-1:/data1
      - ./minio/data/data3-2:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - ./minio/data/data4-1:/data1
      - ./minio/data/data4-2:/data2

  nginx:
    image: nginx:1.19.2-alpine
    hostname: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "9000:9000"
      - "9001:9001"
    depends_on:
      - minio1
      - minio2
      - minio3
      - minio4

  #  jupyter/all-spark-notebook
  notebook:
    container_name: jupyter_notebook
    image: jupyter/all-spark-notebook
    ports:
      - 8888:8888
      - 4040:4040
    environment:
      - PYSPARK_SUBMIT_ARGS=--packages com.amazonaws:aws-java-sdk:1.12.95,org.apache.hadoop:hadoop-client:3.3.1,com.amazonaws:aws-java-sdk-bundle:1.12.95,org.apache.hadoop:hadoop-aws:3.3.1 pyspark-shell
    volumes:
      - ./work:/home/jovyan/work
  # Databases
  pgdb:
    container_name: pg_container
    image: postgres
    restart: always
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: test_db
    ports:
      - "5432:5432"
  pgadmin:
    container_name: pgadmin4_container
    image: dpage/pgadmin4:6.1
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"


```

`nginx.conf`配置文件:
```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  4096;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;

    # include /etc/nginx/conf.d/*.conf;

    upstream minio {
        server minio1:9000;
        server minio2:9000;
        server minio3:9000;
        server minio4:9000;
    }

    upstream console {
        ip_hash;
        server minio1:9001;
        server minio2:9001;
        server minio3:9001;
        server minio4:9001;
    }

    server {
        listen       9000;
        listen  [::]:9000;
        server_name  localhost;

        # To allow special characters in headers
        ignore_invalid_headers off;
        # Allow any size file to be uploaded.
        # Set to a value such as 1000m; to restrict file size to a specific value
        client_max_body_size 0;
        # To disable buffering
        proxy_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 300;
            # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;

            proxy_pass http://minio;
        }
    }

    server {
        listen       9001;
        listen  [::]:9001;
        server_name  localhost;

        # To allow special characters in headers
        ignore_invalid_headers off;
        # Allow any size file to be uploaded.
        # Set to a value such as 1000m; to restrict file size to a specific value
        client_max_body_size 0;
        # To disable buffering
        proxy_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-NginX-Proxy true;

            # This is necessary to pass the correct IP to be hashed
            real_ip_header X-Real-IP;

            proxy_connect_timeout 300;
            
            # To support websocket
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            chunked_transfer_encoding off;

            proxy_pass http://console;
        }
    }
}
```

然后通过`docker compose up -d`启动集群. 在这个 docker compose 文件中同时化部署了PostgreSQL数据库以及PostgreSQL Admin 管理UI。

### MinIO GUI
通过上面的 `docker compose` 命令启动了启动了4个实例的MinIO集群，并通过Nginx进行反向代理, 可以通过 http://localhost:9000 来访问 MinIO 的控制台。

![image.png](https://ae05.alicdn.com/kf/H369e1419546e449c8af79251e88defc5e.png)

### Jupyter Notebook
通过 http://localhost:8888 访问 Jupyter Notebook。
由于访问Nupyter Dashboard首先需要一个token，这里首先需要从 docker logs 中获取该 token

```
docker logs $(docker ps | grep jupyter_notebook | awk '{print $NF}')
```

![image.png](https://ae03.alicdn.com/kf/H536184483fbe48fb8f77d5d8757b2b83G.png)

从 log 中可以看到 `?token=93bc05d6549e689c3409a0ac60b58883c13236aa94245306` 的内容，就可以使用这个 token 来登录了:

![image.png](https://ae05.alicdn.com/kf/Hcb3d4fd18c0d4d44be3e0b35b564d34fb.png)

### PostgreSQL Admin
通过 http://localhost:5050 来访问 PostgreSQL admin dashboard, 通过 "Add New Server" 将 PostgreSQL 添加进来，然后就可以通过 PostgreSQL Admin 来管理数据库了。 
![image.png](https://ae04.alicdn.com/kf/H2a8168b1e50c42208cb3100cf14fca36P.png)


![image.png](https://ae04.alicdn.com/kf/Hecd346ff66d14323877385de6bb99ecbb.png)


## 导入测试数据集
下载上面的电影评分数据集，并在 MinIO 中创建 `bucket1` Bucket，并将解压后的 csv 文件上传到该 bucket 中。
![image.png](https://ae01.alicdn.com/kf/H1d60933a57634f1bae001fe561acbb4bn.png)
总共数据大小在1GB左右。

## 创建 Jupyter Notebook
直接通过JupyterLab创建Notebook, 然后直接在notebook中直接编写python代码来调用Spark进行数据的分析操作。
![image.png](https://ae02.alicdn.com/kf/Hb2558ddfcdc54965a33a5c9dd98411cbY.png)

比如这里调用 Spark 来计算从1加到100的值：
![image.png](https://ae03.alicdn.com/kf/H6c038af6480b4d489dace42ea753fbc87.png)

在 docker compose的 `yaml` 配置中， `notebook` 通过 `environment` 属性配置了 `PYSPARK_SUBMIT_ARGS` 参数，初次运行 Spark 的时候，会检查并下载指定的 packages。
这里主要使用了AWS的 JAVA SDK，通过Amazon S3 API 同 MinIO 进行通信。

```yaml
environment:
- PYSPARK_SUBMIT_ARGS=--packages com.amazonaws:aws-java-sdk:1.12.95,org.apache.hadoop:hadoop-client:3.3.1,com.amazonaws:aws-java-sdk-bundle:1.12.95,org.apache.hadoop:hadoop-aws:3.3.1 pyspark-shell
```

相应的Spark的Job可以通过 http://localhost:4040 页面查看:
![image.png](https://ae05.alicdn.com/kf/H5c11c45d97784f638e016c135943e1bfT.png)

## 分析 MovieLens 数据集

下面就通过Spark来分析MovidLens的数据集。

主要是从MinIO读取数据集中的CSV文件，注册成为table，通过SQL查询出top 100的电影，然后将分析结果保存到MinIO以及PostgreSQL数据库。

### 初始化 SparkSession

在读取CSV文件之前，首先需要创建 `SparkSession` 的实例 `spark`，由于Spark主要是内存计算，这里通过`spark.driver.memory`配置参数配置两个内存，否则运行过程中会产生OOM的问题。

```
from pyspark.sql import SparkSession
spark = SparkSession.builder.config("spark.driver.memory", "5g").getOrCreate()
```

### 配置S3/MinIO连接信息

```python
spark.sparkContext._jsc\
.hadoopConfiguration().set("fs.s3a.access.key", "minio")
spark.sparkContext._jsc\
.hadoopConfiguration().set("fs.s3a.secret.key", "minio123")
spark.sparkContext._jsc\
.hadoopConfiguration().set("fs.s3a.endpoint", "http://192.168.0.9:9000")
spark.sparkContext._jsc\
.hadoopConfiguration().set("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
spark.sparkContext._jsc\
.hadoopConfiguration().set("spark.hadoop.fs.s3a.path.style.access", "true")
spark.sparkContext._jsc\
.hadoopConfiguration().set("fs.s3a.multipart.size", "104857600")
```

### 读取 MinIO CSV 文件

```python
ratings = spark.read\
.option("header", "true")\
.option("inferSchema", "true")\
.csv("s3a://bucket1/ratings.csv")
ratings.createOrReplaceTempView("ratings")
ratings.show()
```

CSV中的内容则以表格的形式显示如下：

```
+------+-------+------+----------+
|userId|movieId|rating| timestamp|
+------+-------+------+----------+
| 1| 307| 3.5|1256677221|
| 1| 481| 3.5|1256677456|
| 1| 1091| 1.5|1256677471|
| 1| 1257| 4.5|1256677460|
| 1| 1449| 4.5|1256677264|
| 1| 1590| 2.5|1256677236|
| 1| 1591| 1.5|1256677475|
| 1| 2134| 4.5|1256677464|
| 1| 2478| 4.0|1256677239|
| 1| 2840| 3.0|1256677500|
| 1| 2986| 2.5|1256677496|
| 1| 3020| 4.0|1256677260|
| 1| 3424| 4.5|1256677444|
| 1| 3698| 3.5|1256677243|
| 1| 3826| 2.0|1256677210|
| 1| 3893| 3.5|1256677486|
| 2| 170| 3.5|1192913581|
| 2| 849| 3.5|1192913537|
| 2| 1186| 3.5|1192913611|
| 2| 1235| 3.0|1192913585|
+------+-------+------+----------+
only showing top 20 rows
```

再次读取`movies`CSV

```python
movies = spark.read\
.option("header", "true")\
.option("inferSchema", "true")\
.csv("s3a://bucket1/movies.csv")
movies.registerTempTable("movies")
movies.show()

+-------+--------------------+--------------------+
|movieId| title| genres|
+-------+--------------------+--------------------+
| 1| Toy Story (1995)|Adventure|Animati...|
| 2| Jumanji (1995)|Adventure|Childre...|
| 3|Grumpier Old Men ...| Comedy|Romance|
| 4|Waiting to Exhale...|Comedy|Drama|Romance|
| 5|Father of the Bri...| Comedy|
| 6| Heat (1995)|Action|Crime|Thri...|
| 7| Sabrina (1995)| Comedy|Romance|
| 8| Tom and Huck (1995)| Adventure|Children|
| 9| Sudden Death (1995)| Action|
| 10| GoldenEye (1995)|Action|Adventure|...|
| 11|American Presiden...|Comedy|Drama|Romance|
| 12|Dracula: Dead and...| Comedy|Horror|
| 13| Balto (1995)|Adventure|Animati...|
| 14| Nixon (1995)| Drama|
| 15|Cutthroat Island ...|Action|Adventure|...|
| 16| Casino (1995)| Crime|Drama|
| 17|Sense and Sensibi...| Drama|Romance|
| 18| Four Rooms (1995)| Comedy|
| 19|Ace Ventura: When...| Comedy|
| 20| Money Train (1995)|Action|Comedy|Cri...|
+-------+--------------------+--------------------+
only showing top 20 rows
```



### 计算 top 100

```python
top_100_movies = spark.sql("""
SELECT title, AVG(rating) as avg_rating
FROM movies m
LEFT JOIN ratings r ON m.movieId = r.movieID
GROUP BY title
HAVING COUNT(*) > 100
ORDER BY avg_rating DESC
LIMIT 100
""")

top_100_movies.show()
# Spark Processing
+--------------------+------------------+
| title| avg_rating|
+--------------------+------------------+
|Planet Earth II (...|4.4865181711606095|
| Planet Earth (2006)| 4.458092485549133|
|Shawshank Redempt...| 4.424188001918387|
|Band of Brothers ...| 4.399898373983739|
|Black Mirror: Whi...| 4.350558659217877|
| Cosmos| 4.343949044585988|
|The Godfather Tri...| 4.339667458432304|
|Godfather, The (1...| 4.332892749244713|
|Usual Suspects, T...| 4.291958829205532|
| Black Mirror| 4.263888888888889|
|Godfather: Part I...|4.2630353697749195|
|Last Year's Snow ...| 4.261904761904762|
|Schindler's List ...| 4.257501817775044|
|Seven Samurai (Sh...|4.2541157909178215|
|Over the Garden W...| 4.244031830238727|
|Sherlock - A Stud...| 4.23943661971831|
| 12 Angry Men (1957)| 4.237075455914338|
|Blue Planet II (2...| 4.236389684813753|
| Rear Window (1954)| 4.230798598634567|
| Fight Club (1999)| 4.230663235786717|
+--------------------+------------------+
only showing top 20 rows
```

### 保存 top 100 到 MinIO

```python
top_100_movies.write.parquet("s3a://bucket1/results/top_100_movies")
```
![image.png](https://ae02.alicdn.com/kf/H37b38c345b8e4eaea4059fcef2b38b860.png)

当运行结束后，可以通过MinIO控制台查看，对应的top 100的数据，已经保存到了MinIO中了。

![image-20211029152355877.png](https://ae02.alicdn.com/kf/H5de33ed814854ae5a7e942e6307f4d28r.png)

### 从MinIO读取Parquet

```python
spark.read.parquet("s3a://bucket1/results/top_100_movies").show()

+--------------------+------------------+
| title| avg_rating|
+--------------------+------------------+
|Planet Earth II (...|4.4865181711606095|
| Planet Earth (2006)| 4.458092485549133|
|Shawshank Redempt...| 4.424188001918387|
|Band of Brothers ...| 4.399898373983739|
|Black Mirror: Whi...| 4.350558659217877|
| Cosmos| 4.343949044585988|
|The Godfather Tri...| 4.339667458432304|
|Godfather, The (1...| 4.332892749244713|
|Usual Suspects, T...| 4.291958829205532|
| Black Mirror| 4.263888888888889|
|Godfather: Part I...|4.2630353697749195|
|Last Year's Snow ...| 4.261904761904762|
|Schindler's List ...| 4.257501817775044|
|Seven Samurai (Sh...|4.2541157909178215|
|Over the Garden W...| 4.244031830238727|
|Sherlock - A Stud...| 4.23943661971831|
| 12 Angry Men (1957)| 4.237075455914338|
|Blue Planet II (2...| 4.236389684813753|
| Rear Window (1954)| 4.230798598634567|
| Fight Club (1999)| 4.230663235786717|
+--------------------+------------------+
only showing top 20 rows
```

### 保存数据到PostgreSQL数据库

上面将分析的数据保存到了MinIO文件系统，下面则将 top 100 的电影数据保存到 PostgreSQL 数据库。

#### 安装pipy包

为了访问PostgreSQL数据库，首先需要安装相应的`psycopg2-binary`和`sqlalchemy`python包。

```python
pip install -i https://pypi.doubanio.com/simple/ --trusted-host pypi.doubanio.com psycopg2-binary sqlalchemy
```

#### 保存到数据库

```python
import psycopg2
import pandas as pd
from sqlalchemy import create_engine

top_100_df = top_100_movies.toPandas()

# Create SQLAlchemy engine
engine = create_engine("postgresql+psycopg2://root:root@192.168.0.9:5432/test_db?client_encoding=utf8")
# Save result to the database via engine
top_100_df.to_sql('test_table', engine, index=False, if_exists='replace')
```

执行完成之后，就可以在PGAdmin中查看结果了。

![image.png](https://ae04.alicdn.com/kf/H72781a0906b349eea27a31001a9a16b3A.png)



#### 读取PostgreSQL数据

上面是保存到数据库，同样的也可以读取数据库中的表数据，然后加载到Spark中进行分析

```python
import psycopg2
import pandas as pd
from pyspark.sql import SparkSession
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2://admin:admin@192.168.0.9:5432/test_db?client_encoding=utf8")
pdf = pd.read_sql('select * from test_table', engine)

# Convert Pandas dataframe to spark DataFrame
df = spark.createDataFrame(pdf)
print(df.schema)
df.show()

StructType(List(StructField(title,StringType,true),StructField(avg_rating,DoubleType,true)))
+--------------------+------------------+
| title| avg_rating|
+--------------------+------------------+
|Planet Earth II (...|4.4865181711606095|
| Planet Earth (2006)| 4.458092485549133|
|Shawshank Redempt...| 4.424188001918387|
|Band of Brothers ...| 4.399898373983739|
|Black Mirror: Whi...| 4.350558659217877|
| Cosmos| 4.343949044585988|
|The Godfather Tri...| 4.339667458432304|
|Godfather, The (1...| 4.332892749244713|
|Usual Suspects, T...| 4.291958829205532|
| Black Mirror| 4.263888888888889|
|Godfather: Part I...|4.2630353697749195|
|Last Year's Snow ...| 4.261904761904762|
|Schindler's List ...| 4.257501817775044|
|Seven Samurai (Sh...|4.2541157909178215|
|Over the Garden W...| 4.244031830238727|
|Sherlock - A Stud...| 4.23943661971831|
| 12 Angry Men (1957)| 4.237075455914338|
|Blue Planet II (2...| 4.236389684813753|
| Rear Window (1954)| 4.230798598634567|
| Fight Club (1999)| 4.230663235786717|
+--------------------+------------------+
only showing top 20 rows
```

## 总结

上面显示了通过 Apache Spark/PySpark, MinIO来分析MovieLens数据集，得益于Docker, Jupyter Docker Stacks等基础工具，从构建环境，到使用Jupyter笔记本、Python、Spark和PySpark开始学习和执行数据分析相当的容易，并且还可以在堆栈中添加额外的容器，如MySQL、MongoDB、RabbitMQ、Apache Kafka和Apache Cassandra。构建一个更为完备的分析平台。

