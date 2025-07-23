---
layout: post
title: How do you know your data is correct?
categories: engineering
---

**TL;DR**
I built a separate validation pipeline that hits the same API as my main ingestion pipeline, but instead of inserting data, it only computes metrics like `count(transaction_id)`, `sum(payout)`, and `sum(revenue)` per hour (based on event datetime). It compares these to what's in the data warehouse and stores the *deltas* in a validation table.

If the deltas are all zero — great. If not, I know exactly when and where things went wrong.  
This approach helps catch silent bugs, data loss, or logic errors — and gives me real confidence that my data is correct.


## Introduction
We often trust our pipelines a bit too much.  

You build a solid ingestion system — it fetches data from an API, processes it, and loads it into your data warehouse. You wrap it in try-catch blocks, maybe add retries. Everything "works." But the truth is, **just because it ran doesn’t mean it did the right thing**.  

This post is about how I built a separate validation pipeline to check my main ingestion process — and why it was worth it.


## The Problem

Let’s say you have a pipeline that:

- Makes requests to an API
- Transforms and processes the data
- Upserts it into a target table

This sounds simple, but…  
- What if a silent bug drops some rows?  
- What if the upsert logic breaks on a specific condition?  
- What if the transformation maps a field incorrectly?

It’s easy to miss these — especially when nothing crashes.


## The Solution: A Second Pipeline for Validation

I decided to build another pipeline. Not to ingest data again, but to verify the data was correctly loaded.

Both pipelines hit the same API endpoint, but the validation pipeline doesn’t touch the warehouse. Instead, it:

1. Pulls API data for a specific time range (e.g., 22:00–22:59)
2. Buckets the data by its **`datetime` field**, not ingestion time
3. Computes:
   - `count(transaction_id)` — the data has unique IDs
   - `sum(payout)`, `sum(revenue)`, and other key metrics
4. Compares these metrics against the corresponding metrics from the target warehouse table
5. Stores the **differences (deltas)** in a separate validation table, partitioned by 1-hour windows

If the delta = 0 → great  
If there’s a mismatch → something’s off


## Why This Works

This strategy gives me a lot of confidence because:

- **Time attribution is accurate**: I group by the actual event timestamp, not when the job ran
- **Deduplication is unnecessary**: the `transaction_id` is unique and enforced
- **It’s efficient**: I store only the differences per hour, not full data dumps
- **It’s traceable**: I can go back and recheck specific hours if something breaks downstream

I can quickly answer questions like:

- Did we ingest all the data from 3pm – 4pm yesterday?
- Why does today’s revenue look low?
- Was there a schema bug at midnight?


## What It Doesn’t Catch (And How to Go Further)

While this works really well, I also thought about what I could be missing. Here are a few optional ideas I’ve considered or plan to add:

### Schema Drift Checks  
Even if counts match, field types or names might change silently.

**Fix:** Log and diff field lists + data types across runs


### Outlier Detection  
What if the API sent bad values, but the row count is right?

**Fix:** Add Z-score anomaly checks on revenue, payout, etc.


### Simple Dashboard  
Raw deltas are powerful — but a quick UI helps debug faster.

**Fix:** A simple dashboard showing 1-hour blocks and validation status


## Final Thoughts

This setup might sound like overkill, but it gives me peace of mind. The data I work with drives reporting, billing, and business decisions. A 10% undercount isn’t just a bug — it’s a breach of trust.

By decoupling ingestion from validation and verifying everything at the source, I’m not just hoping for correctness — I’m measuring it.

---

**You don’t need a 100% perfect system to trust your data —  
but you *do* need a system that tells you when something’s wrong.**

And that’s what this validation pipeline gave me.

