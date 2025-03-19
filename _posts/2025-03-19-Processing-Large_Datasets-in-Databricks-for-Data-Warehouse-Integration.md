---
layout: post
title: Processing Large Datasets in Databricks for Data Warehouse Integration
categories: engineering
---

### Introduction
In an ideal world, all incoming data files would have the same schema, be well-organised, and follow a consistent structure. However, in reality, data files vary in format due to:  

- Differences in schema (some files have different column names).  
- Inconsistent headers (some files have headers, others don’t).  
- Varying numbers of columns across files.  
- Different date formats used across files.  

These inconsistencies make it difficult to process data in a straightforward manner.  
This document outlines a systematic approach to handling such challenges using Databricks and AWS S3 as part of a data warehousing pipeline.  

#### 1. Data Storage and File Tracking
Before processing the files, we first store them in AWS S3.
To track the files, we create a metadata table in Databricks, containing basic file information such as:
File name  
Schema (list of columns for each file)  

Step 1.1: Listing Files in S3  
We use Databricks utilities to retrieve a list of files from the S3 bucket:  

```python
files = dbutils.fs.ls("s3://your_data/path/")  # List all files in the S3 path
file_paths = [f.path for f in files]  # Extract file paths
print(file_paths)
print(len(file_paths))
```

Step 1.2: Extracting Column Names for Each File  
We use parallel processing (via ThreadPoolExecutor) to read each file, infer its schema, and store the column names.  

``` python
from concurrent.futures import ThreadPoolExecutor

def get_file_columns(file_path):
    try:
        df = spark.read.format("csv") \
            .option("header", "true") \
            .option("inferSchema", "true") \
            .option("delimiter", ",") \
            .load(file_path)
        
        df = df.dropna()  # Drop null rows if header rows are missing
        return (df.columns, file_path)  # Return column names and file path
    
    except Exception as e:
        print(f"Failed to read {file_path}: {e}")
        return ([], file_path)

# Run in parallel to maximise efficiency
with ThreadPoolExecutor(max_workers = 8) as executor:
    file_columns_array = list(executor.map(get_file_columns, file_paths))

# Convert results into a Spark DataFrame
column_df = spark.createDataFrame(file_columns_array, ["column_name", "path"])

# Save as a table for reference
column_df.write.mode("overwrite").saveAsTable("table_path.file_column_table")
```
Now, file_column_table contains the file path and the schema (column names) for each file.

#### 2. Standardizing and Normalizing Column Names
Once we have collected column names, we need to normalise them to ensure consistency across all files. This step includes:  

- Unifying variations of column names (e.g., "Fname", "First Name", "firstname" → "first_name").  
- Detecting data types such as email, phone, IP address, and dates.  
- Standardizing column names to a predefined format.  

Step 2.1: Extracting Column Names and Preparing for Normalization  
```python
df = spark.sql(
    """
    with get_data as (
        select 
            regexp_extract(path, r'(.*.\/\/)(folder1\/)(folder2\/)(.*)', 4) as file_name,
            column_name
        from file_column_table
    ),
    combine as (
        select
            concat(array(file_name), column_name) as data
        from get_data
    )
    select *
    from combine
    """
)
df.show(truncate=False)
```

This extracts file names and column names into a structured list.  

```python
data = df.select("data").collect()
list1 = [row["data"] for row in data]
```

Step 2.2: Normalizing Column Names  
We apply rules to unify column names:  

```python
import re
from datetime import datetime

def get_column_names(value):
    value = value.strip()

    # Standardise common column name variations
    if value in ("Email", "EMail", "email", "EmailAddress"): return "email"
    elif value in ("Fname", "F Name", "firstname", "FirstName"): return "first_name"
    elif value in ("Lname", "L name", "lastname", "LastName"): return "last_name"
    elif value == "Name": return "name"
    elif re.match("City.*", value): return "city"
    elif value in ("State", "Region", "state"): return "state"
    elif re.match("Country.*", value): return "country"
    elif value in ("Zip", "Postal Code", "zip"): return "zipcode"
    elif value in ("Tel", "Phone", "Phone Number", "phone"): return "phone_number"
    elif value in ("Gender", "Sex"): return "gender"
    elif value in ("DOB", "DateOfBirth"): return "dob"
    elif re.match(".*URL.*", value): return "optin_site"
    elif value in ("IP Address", "IP", "ip"): return "ip_address"
    elif value in ("Date/time stamp", "Date Added", "date_created", "timestamp"): return "optin_datetime"

    # Detect email, phone, and IP formats
    if re.match(r'^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$', value):
        return "email"
    
    if re.match(r'^(\+\d{1,3}\s?)?(\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4})$', value):
        return "phone_number"

    if re.match(r'^(?:\d{1,3}\.){3}\d{1,3}$', value):
        return "ip_address"

    return "unknown"
```

Step 2.3: Applying Normalization
```python
new_values = [[get_column_names(value) for value in row] for row in list1]
print(new_values)
```

This results in a standardised column name mapping.  

#### 3. Storing Processed Schema in S3
To allow manual verification and future processing, we upload the processed schema to S3.  

```python
import csv
from io import StringIO
import boto3

access_key = dbutils.secrets.get(scope="your_scope", key="your_aws_key_name")
secret_key = dbutils.secrets.get(scope="your_scope", key="your_aws_secret_key_name")

s3 = boto3.client("s3", aws_access_key_id=access_key, aws_secret_access_key=secret_key)

def upload_list_to_s3(data_list, bucket_name, s3_file_path):
    csv_buffer = StringIO()
    csv_writer = csv.writer(csv_buffer, escapechar="\\", quoting=csv.QUOTE_MINIMAL)
    csv_writer.writerows(data_list)
    
    s3.put_object(Bucket=bucket_name, Key=s3_file_path, Body=csv_buffer.getvalue())
    print(f"File uploaded successfully to s3://{bucket_name}/{s3_file_path}")

upload_list_to_s3(new_values, "your_bucket_name", "your_path/your_file_name.csv")
```

Create a stage table
```python
from pyspark.sql.types import *
from pyspark.sql.functions import *

s3_path = "s3://your_bucket/your_file_name.csv"

schema = StructType([
    StructField("check", IntegerType(), True),
    StructField("file_name", StringType(), True),
    StructField("columns", StringType(), True)
])

df = spark.read.format("csv").option("inferSchema", "true").schema(schema).load(s3_path)

df = df.withColumn("columns", split(col("columns"), ","))

df.show()
df.printSchema()

spark.sql("drop table if exists stage_table")
df.write.mode("overwrite").saveAsTable("stage_table")
```

#### 4. Importing Data into Databricks Tables
Once columns are normalised, we load the actual data.  

```python
df = spark.sql("SELECT * FROM stage_table")
data_dict = {row["file_name"]: row["columns"] for row in df.collect()}

files = dbutils.fs.ls("s3://path/to/your/data/")
required_fields = ["col_w", "col_x", "col_y", "col_z"]

for file in files:
    schema = data_dict.get(file, [])
    if not schema:
        continue

    df = spark.read.csv(f"s3://your_bucket/destination_folder/{file}", schema=schema)
    df.write.mode("append").saveAsTable("main_table")

df = spark.sql("SELECT * FROM main_table")
df.show(truncate=False)
```

### Conclusion
This approach ensures scalability, consistency, and efficiency.  

- Scalability – Parallelised schema extraction speeds up processing.
- Consistency – Standardised column names create a structured dataset.
- Efficiency – Data is directly stored in Databricks for easy querying.

This workflow provides a robust foundation for importing heterogeneous datasets into a data warehouse.
