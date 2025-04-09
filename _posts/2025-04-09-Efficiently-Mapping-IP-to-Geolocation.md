---
layout: post
title: Efficiently Mapping IP Addresses to Geolocation
categories: engineering
---

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
```sql
WHERE ip_integer BETWEEN start_ip AND end_ip
```
This worked beautifully for IPv4 addresses.  

