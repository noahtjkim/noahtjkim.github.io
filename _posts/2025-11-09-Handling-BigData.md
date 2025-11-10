---
layout: post
title: How do you make your pipeline faster for handling big data?
categories: engineering
---

**TL;DR**  
Fine tune a little bit on your pipeline to boost the data pipeline performance.  

`spark.sql.files.maxPartitionBytes` default is 128MB.  
`spark.hadoop.fs.s3a.connection.maximum` default is 15 - 50.  
`spark.hadoop.fs.s3a.threads.max` default is 10 for old, 96 (Modern Hadoop AWS perf guide).  
`spark.hadoop.fs.s3a.readahead.range` default is 64KB.  
`spark.sql.adaptive.enabled` default `true`, older Spark 3.0 is `off` by default.  

When your pipeline starts reading massive files (in my case, **150 GB+**), you quickly realize that it’s **not Spark being slow** — it’s about how Spark **reads data** and **talks to Amazon S3**.

In one of my pipelines, the total runtime dropped from **8 minutes 8 seconds → 1 minute 5 seconds** —  
a **7.5× speed-up** — without changing a single line of business logic.  

All I did was fine-tune five key configurations that control **partitioning**, **concurrency**, and **I/O behavior**.

Let’s go through what each configuration does and how it helped.

---

### 1. `spark.sql.files.maxPartitionBytes = 512MB`

The default partition size in Spark is **128 MB**. That means Spark splits your dataset into 128 MB chunks before assigning them as tasks to executors.

For a **150 GB file**, that’s over **1,100 partitions**.  
Each partition triggers a separate task, which means:
- More scheduling overhead,
- More shuffle operations,
- And a longer overall runtime.

To fix this, I increased the value to **512 MB**.  

Why 512 MB?  
Because each **worker node** in my cluster has **32 GB RAM and 4 cores**, giving roughly **8 GB per core** to work with.  
Handling a 512 MB partition per task is well within that limit, even with Spark’s memory overhead.

**Effect:**  
Larger partitions → fewer tasks → fewer shuffle operations → less overhead → faster job.

---

### 2. `fs.s3a.connection.maximum = 500`  
### 3. `fs.s3a.threads.max = 500`

These two settings work hand-in-hand.  
They define how Spark’s **S3A client** (the underlying connector for S3) manages **parallel network communication**.

- **`connection.maximum`** controls how many simultaneous connections can be opened to S3.  
  - The default is often as low as **15–50**.
- **`threads.max`** defines how many threads Spark can use to process those connections.  
  - Defaults can be **10–96**, depending on the Hadoop version.

In my setup, I set **both to 500**, meaning Spark can open **500 concurrent S3 connections**, and each connection is backed by a dedicated thread.

**Effect:**  
All executors can now download partitions in parallel instead of waiting for limited network slots.  
This fully saturates available network bandwidth and eliminates connection starvation.

---

### 4. `fs.s3a.readahead.range = 16MB`

This setting controls how much data Spark **pre-fetches per HTTP GET request** from S3.  

By default, it’s **64 KB**, which means Spark issues a new request every 64 KB of data.  
That’s fine for small files, but terrible for 150 GB datasets — you end up making **millions of small GET requests**.

I increased this to **16 MB**, so Spark fetches data in much larger chunks.

Let’s say your file is 150 GB:
- With 64 KB readahead → about **2.4 million GET requests**.
- With 16 MB readahead → only about **9,600 GET requests**.

That’s a **250× reduction** in HTTP requests!

**Effect:**  
Fewer requests → less overhead per partition → higher throughput → faster I/O.

---

### 5. `spark.sql.adaptive.enabled = true`

Adaptive Query Execution (AQE) lets Spark optimize queries **dynamically at runtime**.  

Once enabled, Spark can:
- **Merge tiny partitions** that are created after shuffles or filters,
- **Handle data skew** more gracefully,
- **Adjust join strategies** on the fly.

This means even if your initial partitioning isn’t perfect, Spark adapts to real-world data sizes during execution.

**Effect:**  
Reduces the impact of skew and small fragments — Spark automatically balances workloads and keeps resources efficiently used.

---

### Summary: How It All Comes Together

| Config | Default | New | Effect |
|:--|:--:|:--:|:--|
| `spark.sql.files.maxPartitionBytes` | 128 MB | 512 MB | Fewer partitions, less shuffle |
| `fs.s3a.connection.maximum` | 15–50 | 500 | More concurrent S3 connections |
| `fs.s3a.threads.max` | 10–96 | 500 | Enough threads for all connections |
| `fs.s3a.readahead.range` | 64 KB | 16 MB | Fewer HTTP requests |
| `spark.sql.adaptive.enabled` | false/true | true | Merges small partitions dynamically |

When combined:
- Larger partitions reduce Spark scheduling overhead.  
- High concurrency keeps network usage fully optimized.  
- Bigger readahead drastically cuts down HTTP request counts.  
- AQE merges leftover small partitions at runtime.

As a result, the pipeline loads data **from S3 to Spark up to 7.5× faster** —  
transforming what was once an 8-minute job into a 1-minute task.

---

### Final Takeaway

Big data optimization isn’t about rewriting your code.  
It’s about understanding **how Spark reads data** and **how your infrastructure behaves under load**.

Once you align **partition size**, **network concurrency**, and **I/O behavior** with your cluster’s hardware, Spark stops “being slow” — and starts performing like the distributed engine it was designed to be.

> **Lesson learned:** You don’t need more compute — you need smarter configurations.
