# Hybrid Index: Reducing the Storage Overhead of Main-Memory OLTP Databases

>Lingze
>
>[Paper](https://15721.courses.cs.cmu.edu/spring2020/papers/09-compression/zhang-sigmod2016.pdf)

This paper propose a hybrid index to compress the memory usage at the expense of small performance. For traditional index design, they treat all of the data in the underlying table equally. However, this assumption is incorrect for many OLTP application. The truth is that some tuples are accessed and modified frequently called hot data, others are looked up only called cold data.  

According to the workload of certain application,  it divide the index into two part, the dynamic index and static index. The dynamic part is small and served for update and insert operation. The static part is bigger in which data is compressed. 



This Hybrid Index is named *Dual-Stage Hybrid Index Architecture*, shown as (It is easier to understand the whole process)

![Overview](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HybridIndex/overview.png)



### Convert to Dual-Stage Index.

There are four types of index listed in paper, B+tree, Masstree, SkipList, ART. For certain index, people should manually design a method to convert to the static stage in Dua-Stage Hybrid index. 

This paper propose a general steps:

- Compaction
- Structural Reduction
- Compression.

Considering the data appended in future, the common index will leave some memory for them. However, in static stage, these cold data is considered read-only. So the static part, we don't have to maintain extra memory for insert operation. *Compaction* is to compact these unused memory.

*Structural Reduction* is to remove pointers and other elements that are unnecessary for read-only operation. removing these pointers and instead using a single array of entries that are stored contiguously in memory saves space and speeds up linear traversal of index.

*Compression* is to compress internal nodes or memory pages used in index. Use Snappy or LZ4 algorithm to compress. Actually, only B+tree is the one of above four index to execute Compression steps in the experiment of this paper.

The fig show these how to convert these four index to the static stage part in Dual-stage Hybrid index.

![StaticConvert](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HybridIndex/StaticConvert.png)



### Merge from dynamic part to static part.

This paper discuss two design decision: what to merge, and when to merge.

what to merge:

1. all merge,  move all index in dynamic part to static part.
2. merge cold data. It seems reasonable. Track key popularity and selectively meges the cold entries to the static stage.

However, how to track the cold data is not descriped in detail.

when to merge, still two  strategies

1. set a certain threshold size for dynamic part. (constant trigger)
2. the size of the dynamic part is adaptive as certain percentages of the size of static part.

The  advantage of ratio-based trigger is that it automatically adjusts the merge frequency according to the index size, which can prevent from merging too frequentlty.

Constant trigger works well for read-intensive workloads because it bounds the size of the dynamic stage ensuring fast look-ups.

**Personally speaking, this part is not necessary **



### Experiement

It shows that for read-only txn, the compact index's performance is even better than normal index. For compressed index, B+tree, it is acceptable that the performance drop in, since the cost of compress and uncompress process.

![ReadPerformance](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HybridIndex/ReadPerformance.png)

The overview of detailed experiments. Actually, the compression step is redundant. The compact index can get a good trade-off between memory saving and throughput performance. The compact index can even perform better in read/write txn and read-only txn, though the performance collapse in insert-only and scan/insert txns. 

![PrimaryExp](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HybridIndex/PrimaryExp.png)

In in-memory system experiment, for article dataset, the memory saving is small, about 20%. However, the performance drop slightly. 

![SystemBench](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HybridIndex/SystemBench.png)

In out-disk experiment, this paper proposes a point that for a certain amount of memory,  hybrid index save more memory means more entries can be store in memory. Thus, the IO operation from disk is fewer than common index, which help the performance. 

From the initialization period of Hybrid-Compressed Index, we can see that the IO operation of Hybrid-Compressed Index is obviously happened later than Common Index. With the increase entries, the IO operations between disk and memory become more and more, which will collapse the performance of index.

![LargerThanMemory](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HybridIndex/LargerThanMemory.png)



### Summary

This paper offer a idea. However, it's too idealistic. In reality, how to judge the data is cold or hot is unsolve problem. Maybe this idea can be considered as trick idea for certain application. Currently, the development of index in database is served for general senerios.