# emr-hudi-getting-started
emr-hudi-getting-started

![Screenshot 2024-09-04 at 5 32 11 PM](https://github.com/user-attachments/assets/7c6e5f0a-2330-4fb1-b392-34d6c4f6adcf)


# Steps 

### Step 1: Create EMR Serverless(emr-7.1.0) Cluster

### Step 2: Copy Application ID 
```
aws emr-serverless list-applications
```

### Step 3: Create hudi_job.py 
```
from pyspark.sql import SparkSession
from pyspark.sql.types import *

# Initialize Spark session
spark = SparkSession \
    .builder \
    .appName("hudi_cow") \
    .getOrCreate()


# Initialize the bucket
table_name = "people"
base_path = "s3:/XX/datalakes/hudi-dataset"

# Define the records
records = [
    (1, 'John', 25, 'NYC', '2023-09-28 00:00:00'),
    (2, 'Emily', 30, 'SFO', '2023-09-28 00:00:00'),
    (3, 'Michael', 35, 'ORD', '2023-09-28 00:00:00'),
    (4, 'Andrew', 40, 'NYC', '2023-10-28 00:00:00'),
    (5, 'Bob', 28, 'SEA', '2023-09-23 00:00:00'),
    (6, 'Charlie', 31, 'DFW', '2023-08-29 00:00:00')
]

# Define the schema
schema = StructType([
    StructField("id", IntegerType(), False),
    StructField("name", StringType(), True),
    StructField("age", IntegerType(), True),
    StructField("city", StringType(), True),
    StructField("create_ts", StringType(), True)
])

# Create a DataFrame
df = spark.createDataFrame(records, schema)

# Define Hudi options
hudi_options = {
    'hoodie.table.name': table_name,
    'hoodie.datasource.write.recordkey.field': 'id',
    'hoodie.datasource.write.precombine.field': 'create_ts',
    'hoodie.datasource.write.partitionpath.field': 'city',
    'hoodie.datasource.write.hive_style_partitioning': 'true',
    "hoodie.parquet.compression.codec": "gzip",
    "hoodie.datasource.hive_sync.enable": "true",
    "hoodie.datasource.hive_sync.database": "default",
    "hoodie.datasource.hive_sync.table": "hudi_people",
    "hoodie.datasource.hive_sync.partition_extractor_class": "org.apache.hudi.hive.MultiPartKeysValueExtractor",
    "hoodie.datasource.hive_sync.use_jdbc": "false",
    "hoodie.datasource.hive_sync.mode": "hms",
}

# Write the DataFrame to Hudi
(
    df.write
    .format("hudi")
    .options(**hudi_options)
    .mode("overwrite")  # Use overwrite mode if you want to replace the existing table
    .save(f"{base_path}/{table_name}")
)

```

### Upload job on S3  and update ENV VAR
```
aws s3 cp hudi_job.py s3://soumilshah-dev-1995/jobs/hudi_job.py

export APPLICATION_ID="XX"
export BUCKET_HUDI="XX
export IAM_ROLE="arn:aws:iam::XX:role/EMRServerlessS3RuntimeRole"
```
### RUN job
```
aws emr-serverless start-job-run \
    --application-id $APPLICATION_ID \
    --execution-role-arn $IAM_ROLE \
    --job-driver '{
        "sparkSubmit": {
            "entryPoint": "s3://'$BUCKET_HUDI'/jobs/hudi_job.py",
            "sparkSubmitParameters": "--conf spark.jars=/usr/lib/hudi/hudi-spark-bundle.jar --conf spark.serializer=org.apache.spark.serializer.KryoSerializer"
        }
    }' \
    --configuration-overrides '{
        "monitoringConfiguration": {
            "s3MonitoringConfiguration": {
                "logUri": "s3://'$BUCKET_HUDI'/logs/"
            }
        }
    }'


```


