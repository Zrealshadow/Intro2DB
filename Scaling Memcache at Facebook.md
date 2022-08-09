# Scaling Memcache at Facebook

>Author : Lingze
>
>[PaperLink](https://pdos.csail.mit.edu/6.824/papers/memcache-fb.pdf)
>
>It's an experience paper, not about new ideas. (I like it QAQ)

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

+ more memory -efficient (one copy of (k,v))
+ works well if no key is very popular
+ each web server must talk to many memcache server (every key is possible to arise in EFs)

Replication:

- Good if a few keys are very popular. (For one server of all replicates, its overhead become less, since EFs can request to any of replicates server to get resources)
- Fewer TCP connection 
- less total data can be cached (Storage consumption)



