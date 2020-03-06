## Databricks Delta FAQ


#### When should we use OPTIMIZE (Results in 1GB files) – we are currently running optimize on all delta tables, 3 times weekly.
this really depends on your data.   on one hand, you do need to run OPTIMIZE to make sure the data layout is optimal for query performance.   on the other hand OPTIMIZE can be expensive for TB scale data and you don't want to run it more frequently than needed.   

ok, that's like 100% accurate statement but quite useless, isn't it :-).   but the message i want to get across is that there is a trade-off and you need to analyze several factors (table size, update volume, update frequency, query speed SLA etc).      w.r.t your question of when to do it.    here are few suggestions:
a). if the table(s) are small and being queried often (hourly or daily), plus table updates are frequent (potentially causing uneven sized parquets),  then i'd run optimize more aggressively
b). if the table(s) are huge, updates are weekly and small number of records being touched,  then 3 times/wk or 1 times/wk would be OK 
c). a better way or more intelligent (AI) approach would be to add to your pipeline a performance profiling component ----  analyze the size distribution of all parquets of all tables, and if the mean (center of mass) of the size distribution is above 1.5 or 2 (or whatever acceptable threshold) standard deviation of 1GB (that's the expected size of those files), then OPTIMIZE it.  Otherwise, hold off OPTIMIZE
c.2). alternative way to do (c) above is an empirical method.   have a Golden Set of queries that you know you often run against those delta tables.   profile the run time of those queries,  if it is significantly slower (pick a number for being slow) than what your SLA is,  run OPTIMIZE.   Otherwise, hold off OPTIMIZE


#### Should we use Auto-Optimize (Results in 128MB files) for our data lake?
Similar situation here.   AO definitely helps prevent many small size parquets say 500KB or 10MB.  But you pay for auto-optimize at table write time as it can cause more shuffling in order to make the final output parquet size close to the default blocksize of 128mb.   


#### Should we use Auto-Optimize (Results in 128MB files) for our data lake?
Similar situation here.   AO definitely helps prevent many small size parquets say 500KB or 10MB.  But you pay for auto-optimize at table write time as it can cause more shuffling in order to make the final output parquet size close to the default blocksize of 128mb.   

To further complicate this "depends" is that AO needs to be considered in the context of OPTIMIZE above.  If you OPTIMIZE  3 times / week, with daily update frequency to your Delta lake,  i'd say you would probably be ok without AO.    If you implement the algorithms mentioned above as part of your ingestion pipeline, you will not need AO since OPTIMIZE will be applied if deemed necessary algorithmically

My gut feeling (without seeing your tables, queries, or BI tools) is that you probably do not need AO and in return saves on table write speed.   Because i don't recall you have streaming job running, nor do you have business critical BI or Web Service running on top of the Delta Lake.  In other words, i think your Delta Lake can tolerate certain degree of less-optimal data layout caused performance degradation (between you run your OPTIMIZE)

The recommended Opt-in for AO is when    https://docs.microsoft.com/en-us/azure/databricks/delta/optimizations/auto-optimize
When to opt-in
Streaming use cases where minutes of latency is acceptable
When you don’t have regular OPTIMIZE calls on your table




####  Can we run OPTIMIZE at the same time we are inserting/merging data into Delta tables? (currently we are pausing our ingestion during this window)
This is a. great question.  You are really drill deep into the isolation level of Delta tables.   I would recommend to isolate OPTIMIZE and ingestion to avoid write conflicts.  Typically i tell people to run OPTIMIZE at night time after ingestion is completed. 
https://docs.databricks.com/delta/concurrency-control.html
https://docs.databricks.com/delta/optimizations/isolation-level.html

#### What about VACUUM? How often should we be running this?  Is it safe to do this while loading data?

Performance wise, it does not matter.  You run it to save storage.   You can run it as frequently as you'd like but it probably makes sense to coordinate the VACUUM with your ingestion pipeline run cadence.    The thing you want to watch out for are the following
a).  try not to run VACUUM before your data ingestion pipeline completes successfully
b).  effect on the TIME TRAVEL.     There is a default setting that to remove any uncommitted files >7 days.   So if you VACUUM with this default retention setting, then you can travel in time more than 7 days.   So you'd want to consider change that setting according to your data use case. 
https://docs.databricks.com/delta/delta-utility.html



#### One follow up question.  The optimize docs mention that running optimize twice will have no effect.  So is it smart enough on large tables to know which files need optimizing and which files have already been optimized ?   So, is there any downside to running OPTIMIZE more often?  In other words, if you have a large table, since it is idempotent, is there any negative impact to running OPTIMIZE every day?  So, If we are only loading new data (append only) to a very large table, would OPTIMIZE have a negative impact? 

If it is INSERT, it will not.   you can see that from the table listed in the doc site.  it states that OPTIMIZE does not have write conflict with INSERT.     however, if it is MERGE INTO that will be impacted.   let's talk more about your use case next week.    my recommendation is that -- keep it simple.   OPTIMIZE at the end of pipeline,  if you do not need to have OPTIMIZE running at the same time as your data ingestion.      given all the subtlety and complexity, it's better to engineer away from the potential risk in a full Prod scenario.    what do you think?
 
 

#### One side unrelated question – in my researching of this question, I ran across DBIO.  (https://docs.databricks.com/_static/notebooks/dbio-transactional-commit.html).   Could you explain to me more about what DBIO is, and how we can use it?  I found it mentioned in the docs (https://docs.microsoft.com/en-us/azure/databricks/spark/latest/spark-sql/dbio-commit) but it doesn’t provide much information there.

re: DBIO, there are two related things.  DBIO caching is an optimization feature that we provide as part of the product.   the best way to think about it is to make analogy to the instruction prefetching in a computer system architecture design world.   smiliarly, data/partitions are prefetched and cached to speedup remote file transfer

DBIO transactional commit (your questions) concerns with write performance and transactionality.   In a nutshell, it is optimization technique that improve write commit performance (to cloud storage like BLOB or S3) by allowing each task to write out its data more aggressively earlier than previous version (<2.1) spark.   At the same time, it keeps track of transactional commit status making Delta transactional ACID compliant.    VACUUM is designed a big part of it to solve the problem of those early writes but later on uncommitted (similar to rolled-back concept in DBMS work)

last but not least, it does appear that you guys are much more mature in adopting the Delta technology.    i believe we talked about setting up Delta workshop.   you will like it.    i would highly recommend the formal training class DB200 Delta 
https://academy.databricks.com/instructor-led-training/DB200
as i am one of the few people who saw how Delta came along from conceptual idea to full-fledging product,  i can assure you that you will find this fascinating!     just a bit context --- it was at the same time Delta was being developed,  three of us worked with a customer and developed our own version of Delta (only part of the functionality).   it took us 6 months.    so this is really a remarkable technology so easy and simple to use :-) 
