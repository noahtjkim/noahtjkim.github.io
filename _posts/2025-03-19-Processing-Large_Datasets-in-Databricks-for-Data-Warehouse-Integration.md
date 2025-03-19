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

# Run in parallel to maximize efficiency
with ThreadPoolExecutor(max_workers = 8) as executor:
    file_columns_array = list(executor.map(get_file_columns, file_paths))

# Convert results into a Spark DataFrame
column_df = spark.createDataFrame(file_columns_array, ["column_name", "path"])

# Save as a table for reference
column_df.write.mode("overwrite").saveAsTable("table_path.file_column_table")
```
Now, file_column_table contains the file path and the schema (column names) for each file.

#### 2. Standardizing and Normalizing Column Names
Once we have collected column names, we need to normalize them to ensure consistency across all files. This step includes:  

- Unifying variations of column names (e.g., "Fname", "First Name", "firstname" → "first_name").  
- Detecting data types such as email, phone, IP address, and dates.  
- Standardizing column names to a predefined format.  

Step 2.1: Extracting Column Names and Preparing for Normalization
₩₩₩ python
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
₩₩₩


This extracts file names and column names into a structured list.

python
Copy
Edit
data = df.select("data").collect()
list1 = [row["data"] for row in data]
Step 2.2: Normalizing Column Names
We apply rules to unify column names:

python
Copy
Edit
import re
from datetime import datetime

def get_column_names(value):
    value = value.strip()

    # Standardize common column name variations
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
Step 2.3: Applying Normalization
python
Copy
Edit
new_values = [[get_column_names(value) for value in row] for row in list1]
print(new_values)
This results in a standardized column name mapping.


