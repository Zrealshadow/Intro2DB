# Scaling Memcache at Facebook

>Author : Lingze
>
>[PaperLink](https://pdos.csail.mit.edu/6.824/papers/memcache-fb.pdf)
>
>It's an experience paper, not about new ideas. (I like it QAQ)
>
>[Advanced Paper Link](http://cs.cmu.edu/~beckmann/publications/papers/2020.osdi.cachelib.pdf)



### Context

-  single machine w / web server + application + DB, DB provides persistent storage, crash recovery, transactions. As load grows, application takes many CPU Time.
- Many web frontEnd(FE) ,  one shared DB.  Web FE is server + application which is always separated from storaget. FEs are stateless, which can server any request, no harm from FE crash.
- Many web FEs, data sharded.  Good DB parallelism but hard to partition too finely. (DBs are slow, why not cache read request)
- Many web FEs, many caches for reads, many DBs for writes. Cache is 10 faster than a DB.

### Case

features of facebook application scenarios:

- fresh/consistent data is not critical, humans are tolerant
- High Load : billions of operations per second
- Multiple data centers (we call each data center "region")

**In this scenarios, the cache is critical.  not really about reducing user-visible delay. mostly about shielding DB from huge overload.**

The target of this system want to solve these problems:

- want to avoid unbounded staleness ()
- **want to read own writes**
- huge " fan-out" -> paralled fecth, in-cast congestion

![](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Memcache/Cache.png)

Figure 1 shows how to use Memcache in system.

For `get`

- FE get value from memcache server according to Key k
- if hit, return value
- else request value to DB and set (k, v) pair in memcache

For `put`

- FE put (k, v) pair directly to DB
- FE broadcast to memcahce servers that (k, v_) is invalidate.



Performance comes from parallelism

Two basic parallel strategies for storage: partition vs replication.

Partition: divide keys over memcache servers

replication: divide clients over memcache servers.

Partition:

+ more memory-efficient (one copy of (k,v))
+ works well if no key is very popular
+ each web server must talk to many memcache server (every key is possible to arise in EFs)

Replication:

- Good if a few keys are very popular. (For one server of all replicates, its overhead become less, since EFs can request to any of replicates server to get resources)
- Fewer TCP connection 
- less total data can be cached (Storage consumption)



### System Overview

![](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Memcache/MemcacheOverview.png)

We skip the content of "region". For detail, check the the section 5 in raw paper.

For one region.

The Storage part and logic processing part are seperated.  The Front-End Cluster is a set of memcache server. 

Different Clusters are replicates (Maybe not correct, one cluster can be regarded as a completely independent request processor unit), while different Memcache servers are partioned in one cluster.

**How to work**

In this system,  adding memcache server into cluster doesn't help single popular key. Adding a cluster dose help.

In cluster, client will talks to all memcache server in parallel, since the data is partioned in all these memcache servers. In this case, all replied from memcache server come back at the same time, which leads to the in-cast congestion mentioned in paper.

**Regional Pool**

replicating is a waste of RAM for less-popular items. So this paper propose a "regional pool", which store unpopular objects and shared by all clusters. In this way, it can avoid replicate many unused data to save the memory consumption.



**New Cluster start up**

When a new cluster starts up, all request assigned to this cluster will not hit the cache. It will generate big spike in DB load. How to solve it, the clients of new cluster first get() from existing cluster and set into this new cluster. (2x load on existing cluster than 30x load on DB).



**If memcache server fails**

DB will have too much load. cann't shift load to other memcache server. cann't re-partition all data

Solution: **Gutter**  -- pool of idle mc servers, clients only use after mc server fails.

Gutter just a emergency server.



### Consistency.

Actually, any distributed system problem should consider how to keep data consistancy between replicate.

The goal to keep consistancy of Memcache system:

- read don't always see the latest write (stale data is tolerant)
- Writers should see their own write.



DB consistancy is keeped based on MySQL replicated mechanism.

**How to keep memcache content consistent with DB content ?**

![](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Memcache/MemcacheMcSqueal.png)

Shown in figure 

Two mechnism :

- DBs send invalidates (Key k is deleted) to all memcache server
- Writing client when put will also invalidates memcache in local cluster for read-your-own-writes.

There some races should be considered.

**Race 1**

Situation

>K not in cache.
>
>​          Client1                                                                                   Client 2
>
>C1 get(k), misses
>
>C1 v1 = read k from DB
>
>​																								C2 writes k = v2 in DB
>
>​																								C2 delete (k)
>
>C1 Set(k, v1) in Cache

now Memcache Server in cluster1 will have stale data.  Delete(k) has already happend. It will stay stale indefinitely until k is next written.

Solution : **Lease**. to keep the step that read k from DB and the step to set is a atomic operation.

C1 gets a lease from mc. now C2 delete(k) in DB. Db will send invalidates to all memcache server. Now the k lease in cluster is invalid. Now C1 should re-read k from DB , get new lease and do above process again.



**Race 2**

Situation, During the cold cluster warm-up.

on miss, client will get in warm cluster, copy to cold cluster.

>K starts with value v1
>
>​             Client 1                                                             Client 2
>
>C1 update k to v2 in DB
>
>C1 delete (k)  -- in cold cluster 
>
>​																					C2 get(k) miss -- in cold cluster
>
>​																					c2 v1 get(k) from warm cluster, hit
>
>​																					c2 set (k, v1) into cold cluster
>
>Delete already happend in cold cluster local memcache, but not happend to warm cluster (through mysql Pipeline).
>
>Thus copy the stale data from warm cluster

Solution : **Two-second hold-off**. Just used on cold clusters. after C1 delete, **cold memcache ignores set for two seconds.** In this two second, delete() will propagate via DB to warm cluster.



**Race 3**

>K starts with value v1
>
>​			Client 1 																	
>
>Client1 is in a secondary region
>
>C1 updates k = v2 in primary DB
>
>C1 delete (k)  -- local region.
>
>C1 get k miss
>
>C1 read local DB --- see v1 not v2. later v2 arrives from primary DB according to mysql replicated mechnism

Solution: **remote mark**

C1 delete(k) marks key "remote", get() miss yield "remote". then these mark will tell the Client1 to read from "Primary DB" instead of local DB. "Remote" cleared when new data arrives from primary region.

(It's a high-level summary, we skip the detail of implementation of "remote mark")





Question:

>Why put a new (k,v) into DB, we have to delete key k in cache instead of updating the new pair ?

- Cache Value is not often simply a literal DB record.
- increase read-your-own writes delay
- DB will send unrelated value to Cache



### Summary

```
PNUTS does take this alternate approach of primary-updates-all-copies

FB/mc lessons for storage system designers?
  cache is vital for throughput survival, not just to reduce latency
  need flexible tools for controlling partition vs replication
  need better ideas for integrating storage layers with consistency
```