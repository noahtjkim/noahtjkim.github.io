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
02/05/24 (mm/dd/yy)
...

What if you have datasets which come from multiple data providers or sources in your S3 and what if their date formats are all different?  
Your mission is to import the datasets to your data warehouse and the date fields should come as timestamp type.  
It will not be easy as long as there are multiple different date formats.  

There should be many ways to handle this case.  
I'd like to introduce one of the ways.  

1. Import the date field as a string type.  
   Then this field will have all date formats.

2. You have to acknowledge what date formats are there.


