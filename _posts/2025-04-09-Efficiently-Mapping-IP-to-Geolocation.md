---
layout: post
title: Efficiently Mapping IP Addresses to Geolocation
categories: engineering
---

**TL;DR**  
How to efficiently map IP addresses to geolocation data without a paid API.  
- Avoid expanding CIDR blocks into individual IPs - it's memory intensive and inefficient.
- Convert IPs to integers (for IPv4) and compare them against CIDR start and end ranges.
- Use libraries like ipaddresses (in Python) to convert CIDR to start/end and IP to integer.
- IPv6 integers are too large for Bigint, so store them as strings and use string comparison.
- Plist IPv4 and IPv6 into separate tables to maintain performance, especially if IPv4 traffic dominates.
  This method helped me build a scalable, cost-efficient, and fast IP geolocation lookup without relying on paid APIs.

When dealing with marketing data, one common challenge is working with IP addresses—especially  
if you want to enrich them with geolocation information.  
My dataset had a mix of IPv4 and IPv6 addresses but lacked any associated geolocation data  
like city, state, or country.

Recently, I obtained a dataset that maps IP blocks to geolocation information.  
Each row in this new dataset included IP blocks (CIDR format) e.g. 192.168.0.0/24, 2001:200::/32  
and corresponding geolocation details like zipcode, city, state, country, latitude, and longitude.  

If budget weren’t a concern, the simplest approach would be to use a paid IP geolocation API  
(like MaxMind or IP2Location).  
But in this case, I wanted to build something that didn’t rely on external paid services.  

Initially, I thought: why not expand all the IPs within each CIDR block and match my IPs against the list?  
This idea works in theory but fails in practice.  
Let’s break it down:  
- A CIDR block like 192.168.0.0/24 contains 256 IP addresses
- My CIDR-to-location mapping dataset had over 5M rows
- Expanding every block would result in billions of individual IPs
- Memory usage and processing time became unmanageable

Clearly, a better strategy was needed.  

Instead of expanding all IPs, I realised each CIDR block can be represented using a start IP and an end IP.  
e.g. 192.168.0.0/24 -> start ip: 192.168.0.0, end ip: 192.168.0.255  
Now, the trick is to check whether a given IP falls between the start and end boundaries.  
But here's the catch. IP addresses stored as strings don't compare properly using SQL range conditions.  
So I had to convert IP addresses to integers for accurate and efficient comparison.  
e.g. 192.168.0.0 -> 3232235521, 192.168.0.10 -> 3232235530, 192.168.0.255 -> 3232235775  

Once converted, checking whether an IP (e.g. 192.168.0.10) falls within a range becomes a simple SQL condition.  
``` sql
WHERE ip_integer BETWEEN start_ip AND end_ip    
```
This worked beautifully for IPv4 addresses.  

Then what about IPv6? IPv6 addresses present a new challenge.  
Their numerical values can be massive—way beyond the limits of traditional data types.  
Take 2001:200::/32 for example: When converted to an integer, it becomes 42540766411282592856903984951653826560  
This number far exceeds the maximum for BIGINT in most SQL engines (including Databricks),  
which is 9,223,372,036,854,775,807  
So you can’t store IPv6 as integers using standard numeric types.  
Instead, I stored the start and end boundaries of IPv6 addresses as strings.  
This allowed me to perform comparisons using string-based logic in SQL.  

For performance optimisation, separate tables for IPv4 and IPv6.  
When I first tried storing both IPv4 and IPv6 in a single table (with boundaries stored as strings),  
performance took a nosedive—even for IPv4 queries.  

That’s because comparing numeric IPs (IPv4) as strings is much slower than comparing integers.  

So I split the data into two separate tables.  
- IPv4 Table: IPs converted to integers (bigint) for fast, efficient range comparisons.
- IPv6 Table: IPs stored and compared as strings, still slower, bu necessary due to data type limits.
Since the vast majority of my traffic is IPv4, this setup gave me a good balance between performance and flexibility.
