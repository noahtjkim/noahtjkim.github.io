---
layout: post  
title: Optimising Range Joins from 140 Trillion Comparisons to Practical Performance  
categories: engineering
---

**TL;DR**  
Range joins like `ip between start_ip and end_ip` are brutally slow on large datasets because they result in O(n x m) comparisons.  
To solve this, I implemented a bucket-wise join strategy.  
1) Convert each ip into a bucket_id.  
2) Map start_ip - end_ip ranges into overlappting bucket_ids.
3) Join on bucket_id first and filter with between later.  
This reduced the query from 140 trillion comparisons to a scalable, fast join, all without losing accuracy.  

**Optimising range joins on IPs**  
Recently, I ran into a query performance nightmare in Databricks.  
I had two large tabels: table_a (~40M rows) stores individual IPs in a column called ip (bigint),  
table_b (~3.5M rows) defines IP ranges using start_ip and end_ip (both in bigint)  
My task was to find all ips in table_a that fall within the IP ranges in table_b.  
My first trial was like this.  
```sql
select *
from table_a as a
  join table_b as b
    on a.ip between b.start_ip and b.end_ip
```
This join never finished. Because it compares every row from table_a to ever row in table_b. Taht's O(n x m) comparisons.  
4M x 3.5M = 140T  

So I had to take a different strategy to achieve my goal and the strategy is to use Bucket-wise Join strategy.  
I bucketed the IPs to reduce the number of comparisons. Here's how.  
I grouped each IP into a bucket by dividin it by 1M.  
```sql
select
  *,
  cast(floor(ip / 1000000) as bigint) as bucket_id
from table_a
```
e.g. IPs from 0 - 999,999 -> bucket 0, IPs from 1,000,000 - 1,999,999 -> bucket 1, etc.  
This groups similar IPs into numeric buckets.  

For table_b, I exploded each group range into all buckets it touches.  
```sql
select
  *,
  explode(
    sequence(
      cast(floor(start_ip / 1000000) as bigint),
      cast(floor(end_ip / 1000000) as bigint)
    )
  ) as bucket_id
from table_b
```
This tells Spark, "Hey, this IP range overlaps with these N buckets. If any IP in table_a lands in one of these, we should check it."  

Then, join on bucket_id.  
With both tables sharing a bucket_id, I rewrote the join.  
```sql
select *
from table_a as a
  join table_b as b
    on a.bucket_id = b.bucket_id
where a.ip between b.start_ip and b.end_ip
```
This drastically narrows down the number of candidate matches. Only IPs within the same bucket get compared, and the where clause ensures results are exact.  

What about False Positives?  
A false positive here is when table_a and table_b have the same bucket_id. But the IP doesn't actually falls within the range.  
These are harmless because the where clause filters them out. Bucketing just scopes the join candidates. It doesn't change the logic.  

How to choose the right bucket size?  
In my case, I used `bucket_id = floor(ip / 1000000)` then why 1,000,000?  

Here's how to reason about it.  
- Total IP range: 4.2B (e.g. from 0 to 4,288,654,416)
- Bucket size = 1,000,000 -> approximately 4,288 buckets.
- That's a good range because it's not too small (which would cause `explode()` to bloat), not too large(which would introduce too many false positives)

I used this general rule `bucket_size = total_range / target_bucket_count`  
A good target is 1,000 to 10,000 buckets depending on your cluster and join complexity.  

Therefore, the query that never finished, it ran in just a few minutes.  
This technique is reusable in any system where one table has point values (IPs, timestamps, prices, etc.)  
and the other table has ranges (IP blocks, time windows, etc)  

In conclusion, bucketing is one of the simplest yet most powerful optimisations for range joins at scale.  You don't need indexing or advanced partitioning.  
Just smart grouping and filtering.  
If you've ever been frustrated by long-running range joins in Spark or SQL, try this approach.  
It saved me hours and trillions of comparisons.  

