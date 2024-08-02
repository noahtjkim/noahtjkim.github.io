---
layout: post
title: Handling date formats in big data
---

When you are given datasets from multiple sources.   
If it has "date" field, then there might be a chance that the date format is all different because the sources provide it in different format.  

I have seen many different date formats like  
02/25/2024 (MM/dd/yyyy)  
03/04/2024 15:23:10 (dd/MM/yyyy HH:mm:ss)  
22-5-2023 (dd-m-yyyy)  
4-21-2024 (M-dd-yyyy)  
04-21-2024 09:40:11 p.m. (MM-dd-yyyy hh:mm:ss a)  
02/05/24 (MM/dd/yy)
...

What if you have datasets which come from multiple data providers or sources in your S3 and what if their date formats are all different?  
Your mission is to import the datasets to your data warehouse and the date fields should come as timestamp type.  
It will not be easy as long as there are multiple different date formats.  

There should be many ways to handle this case.  
I'd like to introduce one of the ways.  

1. Import the date field as a string type.  
   Then this field will have all date formats.

2. You have to know what date formats are there.
   To know all the date formats in your datasets, you could try this way.
   ```  
   select
      case
         when date rlike '\\d{2}\\/\\d{2}\\/\\d{4}\\s\\d{2}:\\d{2}:\\d{2}' then 'MM/dd/yyyy HH:mm:ss'
         when date rlike '\\d{2}-\\d{2}-\\d{4}\\s\\d{2}:\\d{2}:\\d{2}' then 'MM-dd-yyyy HH:mm:ss'
         when date rlike '\\d{1,2}-\\d{2}-\\d{4}' then 'M-dd-yyyy'
         else 'Unknown formats'
      end
   from dataset_table
   ```

3. Once you get all the date formats, then you could take either one of these ways.
   - Add a new column and use update statement
   - create multiple tables and update all the tables and use union  


