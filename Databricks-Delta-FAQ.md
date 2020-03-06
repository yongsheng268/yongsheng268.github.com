## Databricks Delta FAQ


#### When should we use OPTIMIZE (Results in 1GB files) – we are currently running optimize on all delta tables, 3 times weekly.
this really depends on your data.   on one hand, you do need to run OPTIMIZE to make sure the data layout is optimal for query performance.   on the other hand OPTIMIZE can be expensive for TB scale data and you don't want to run it more frequently than needed.   

ok, that's like 100% accurate statement but quite useless, isn't it :-).   but the message i want to get across is that there is a trade-off and you need to analyze several factors (table size, update volume, update frequency, query speed SLA etc).      w.r.t your question of when to do it.    here are few suggestions:
a). if the table(s) are small and being queried often (hourly or daily), plus table updates are frequent (potentially causing uneven sized parquets),  then i'd run optimize more aggressively
b). if the table(s) are huge, updates are weekly and small number of records being touched,  then 3 times/wk or 1 times/wk would be OK 
c). a better way or more intelligent (AI) approach would be to add to your pipeline a performance profiling component ----  analyze the size distribution of all parquets of all tables, and if the mean (center of mass) of the size distribution is above 1.5 or 2 (or whatever acceptable threshold) standard deviation of 1GB (that's the expected size of those files), then OPTIMIZE it.  Otherwise, hold off OPTIMIZE
c.2). alternative way to do (c) above is an empirical method.   have a Golden Set of queries that you know you often run against those delta tables.   profile the run time of those queries,  if it is significantly slower (pick a number for being slow) than what your SLA is,  run OPTIMIZE.   Otherwise, hold off OPTIMIZE


### Should we use Auto-Optimize (Results in 128MB files) for our data lake?
Similar situation here.   AO definitely helps prevent many small size parquets say 500KB or 10MB.  But you pay for auto-optimize at table write time as it can cause more shuffling in order to make the final output parquet size close to the default blocksize of 128mb.   
