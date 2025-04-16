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
