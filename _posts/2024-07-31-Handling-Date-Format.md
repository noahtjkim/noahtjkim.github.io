---
layout: post
title: Handling date formats in big data
---

When you are given datasets from multiple sources.   
If it has "date" field, then there might be a chance that the date format is all different because the sources provide it in different format.  

I have seen many different date formats like  
02/25/2024 (mm/dd/yyyy)  
03/04/2024 (dd/mm/yyyy)  
22-5-2023 (dd-m-yyyy)  
4-21-2024 (m-dd-yyyy)  
04-21-2024 (mm-dd-yyyy)  
...

What if you have datasets which come from multiple data providers or sources in your S3 and what if their date formats are all different?  
Your job is to import the datasets to your data warehouse and the date fields should come as timestamp type.  
Because of the different formats of date fields, it will cause an error and you have to handle with that.  
