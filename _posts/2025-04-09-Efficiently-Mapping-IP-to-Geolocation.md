---
layout: post
title: Efficiently Mapping IP Addresses to Geolocation
categories: engineering
---

When dealing with marketing data, one common challenge is working with IP addresses—especially  
if you want to enrich them with geolocation information.  
My dataset had a mix of IPv4 and IPv6 addresses but lacked any associated geolocation data  
like city, state, or country.

Recently, I obtained a dataset that maps IP blocks to geolocation information. Each row in this new dataset included:
