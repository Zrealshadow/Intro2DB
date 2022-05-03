# Opportunities for Optimism in Contended Main-Memory Multicore Transactions



>[Raw Paper](https://15721.courses.cs.cmu.edu/spring2020/papers/03-mvcc1/p629-huang.pdf)
>
>给Optimism Concurrency Control 提供了一个思路



### Intro

该论文主要是为OCC证明，经过某些优化，OCC不仅仅在Low Contention的情况下有很好的效果，在High Contention情况下也很好的结果。该论文首先在STO这个平台上实现了三种Concurrency Control, OCC, Tictoc, MVCC。然后说了还有其他的因素会影响Performance, 他称这些因素为Basic Factor， 这些因素是除了并发控制算法之外也能影响到Performance的东西，因此在对比过程中需要将这些因素设置为不变量。然后该paper对这三种并发控制算法做了实验对比。发现在某些数据中High Contention的情况下OCC表现得还可以。然后作者提出了，那么既然OCC在Low Contention情况下很棒，那么优化一下High Contention 是不是很棒呢？ 然后该Paper提出了两个Optimization， Commit-time updates 和 Timestamp splitting， 前一个是将自增操作，变为一个write而不是一个read + write。后一个就是在Tuple Lock基础上更加细粒度锁，减少conflict。

总的来说这篇论文有点像一篇 Survey （感觉CC的论文都有点Survey的味道）， 做了一些总结性的工作和实验，给了一些Benchmark的测试，给不同情况下Concurrency Control的选择提供了思路。很棒（15721的论文都不错）





### Problem

感觉没有提出明确的challenge



### Background

该论文的实验在STO system上进行， STOv2实现了三种CC算法，

- OSTO ， 使用OCC算法
- TSTO， 使用TicToc OCC算法
- MSTO，使用MVCC策略

 STOv2 使用的二级索引包括HashMap和一个Masstree Index 来支持 Scan操作（Masstree Index 标记）

![Overview](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/STO/Overview.png)



##### OSTO

Follow Silo OCC protocal

- Execution Time : transaction logic generates read and write
- Commit Time : run 3 phases
  - In phase 1, the OSTO library locks all records in the write set, aborting if deadlock is suspected. After phase 1, the transaction's timestamp is selected as its serialization point.
  - In phase 2, validates that records in the read set have not changed and are not locked by other transactions, aborting on validation failure
  - In phase 3, installs new versions of the records in the write set, updates their timestamps, and release locks.





##### MSTO

MSTO is an MVCC variant based broadly on Cicada, but lacks some of Cicada's advanced features and optimizations.

Protocal

Only for rw Txn

At commit time

- MSTO choose a commit timestamp TSthc with an atomic increment.

- In phase 1, MSTO inserts a new PENDING version with TSthc into each modified record's version chain. (Concurrent transactions that access a PENDING VERSION in their execution phase will spin-wait until the state change)

- In phase 2, MSTO checks the read set: **if any version visible at TSthc differs from the version observed at TSth (Beginning of Transaction's Timestamp), the transaction aborts.** otherwise, MSTO atomically updates the read timestamp on each version v in the read set to Max{v.rts, TSthc} 

  (根据他的描述，我理解，这里实现的MVCC 感觉会比我所认知的MVCC 更加的严格， 他对于RW冲突进行了Abort， 产生读快照（read Snapshot）的时刻是Txn刚刚产生的时候，Commit的时候如果读的数据中的ReadTimestamp改变了，即该Version在Txn 产生到Commit中该数据被别的Txn读了也会Abort。)  简单来说在Txn Creation时产生的ReadView 应和 Txn Commit时产生的ReadView相同，否则Abort。

- In phase 3, MSTO changes its PENDING versions to be COMMITED and enqueues earlier version for garbage collection.



##### TSTO

Use TicToc in place of plain OCC as the CC mechanism.

It dynamically computes transactions’ commit timestamps based on read and write set information. This allows for more flexible transaction schedules than simple OCC, at the cost of more complex timestamp management.





### Basis Factors

该概念在论文中占很大的篇幅，主要是说transaction Processing system不仅仅受限于CC mechanism， 而且收到其他implementation choices的印象。包括

- Contention regulation
- Memory Allocation
- Abort Mechanism
- Index types
- Contention-aware indexes
- Transaction internals
- deadlock avoidance mechanism



**Contention regulation**

Contention regulation avoids repeated chache line invalidations by delaying retry after a transaction experiences a conflict. **Over-eager retry can cause contention collapse; over-delayed retry can leave cores idle.** 

In STOv2, it use *randomized exponential backoff*



**Memory Allocation**

Transactional systems stress memory allocation by allocating and freeing many records and index structures. This is particularly true for MVCC-based systems, where every update allocates memory so as to preserve old versions. Memory allocators can impose hidden additional contention (on memory pools) as well as other overheads.

In STOv2, it use *fast general -purpose scalable memory allocator* as baseline, have experienced good results with rpmalloc.



**Abort mechanism**

 some abort mechanisms impose surprisingly high hidden overheads. C++ exceptions – a tempting abort mechanism for programmability reasons – can acquire a global lock in the language runtime to protect exception-handling data structures from concurrent modification by the dynamic linker.

In sto, it implements aborts using *explicitly-checked return values* instead.



**Index type**

Hashindex, BtreeIndex.



**Contention-aware index**

We recommend implementing contention-aware indexing, either automatically or by taking advantage of static workload properties. Our baselines implement contention-aware indexing by leveraging a side effect of Masstree’s trie-like structure

Certain key ranges in Masstree will never cause phantom-protection conflicts.





##### Summary

The evaluation is shown in figure. We can see that these basic factors have different level of impact on Transaction Processing system's performance.

![BasicFactorEvaluationTable](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/STO/BasicFactorEvaluationTable.png)

![BasicFactorEvaluation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/STO/BasicFactorEvaluation-1568349.png)



When set these basic factor same, and start to evaluate three types of CC mechanism. The result is shown in Figure.

In some dataset, even through it's high contention, the OCC mechanism outperform than MVCC mechanism. 

The result is very interesting. OvO.

![Evaluation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/STO/Evaluation.png)



### High-Contention Optimizations

在这个章节中， 提出了两个Optimization，这里只记录一下我的理解，详细查看原文

Commit-time updates， 将自增操作作为一个updater算子写入。 原始的自增操作是先读再写， 一个read + write。 这里优化成了一个write操作。根据上文介绍的MSTO的MVCC机制在Commit中的Phase2， 就大大减少的read次数，既减少了在Phase 2 abort rate. 

这个Updater算子写入后，会在之后的write过程中被flatten， 具体细节就不赘述。



Timestamp Splitting， 这个优化相比上面的有点尴尬。主要思路就是partion column，这样读写tuple时，可以对一个column group上锁，而不是对整个tuple 上锁，这样减少了conflict。



然后他还提到了，Commit-time updates + Timestamp Splitting 两个Optimization加在一起会发挥更好的作用。

![OptimizationEvaluation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/STO/OptimizationEvaluation.png)





## 总结

这篇论文更多的是在Implementation中提供了指导性的方向，也越发指出了没有最好的CC mechanism，对于不同的workload每个CC都有好坏。重点是如何在Base CC mechanism的基础上做好一些basic factor的优化，至少我是这样认为的。