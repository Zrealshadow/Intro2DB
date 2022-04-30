# Serializable Snapshot Isolation in PostgreSQL

>It is a good paper about Snapshot Isolation and how to implemente it
>
>Helped me deepen my understanding of Snapshot Isolation.
>
>[Link](https://15721.courses.cs.cmu.edu/spring2020/papers/03-mvcc1/p1850_danrkports_vldb2012.pdf)
>
>Following cmu15721

Serializable isolation for transactions is an important property, it can force transactions to execute sequentially. However the standard two-phase locking mechanism which gaurantee Seiralizable isolation is too expensive. So people want to use snapshot isolation, which offers greater performance but allows certain anomalies.

This paper gives a new concept **Serializable Snapshot Isolation**. Normal Snapshot Isolation can not handle some anomalies. the authors of this paper analysize these anomalies and give a set of optimization and enhancement based on Snapshot Isolation to avoid the arise of these anomalies.  It also introduce the detailed implementation in these whole setting in PostgreSQL 9.1 version.



#### Snapshot Isolation (IS)

SI is one particular weak isolation level that can be implemented efficiently using multiversion concurrency control, without read locking. SI does not allow the three anomalies defined in the ANSI SQL standard: dirty reads, non-repeatable reads and phantom reads. But, it does not guarantee serializable behavior, it allow certain anomalies. This unexpected transaction behavior can pose a problem for users that demand data integrity.

Two anomalies:

- Simple Write Skew
- Batch Processing



**Write Skew**

Two concurrent transactions that read the same data, but modify disjoint sets of data.

![WriteSkew](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/SSI/WriteSkew.png)

If use 2PL, the execute order will be T1 - T2. the Bob will still on-call. However, in SI, both transactions read from a snapshot taken when they start. they both proceed to remove Alice and Bob, respectively from call status.



**Batch Processing**

![BatchProcessing](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/SSI/BatchProcessing.png)

Consider a transaction-processing system that maintains two tables. A receipts table tracks the day’s receipts, with each row tagged with the associated batch number. A separate control table simply holds the current batch number. 

There are three transaction types: 

​	• NEW-RECEIPT: reads the current batch number from the control table, then inserts a new entry in the receipts table tagged with that batch number 

​	• CLOSE-BATCH: increments the current batch number in the control table 

​	• REPORT: reads the current batch number from the control table, then reads all entries from the receipts table with the previous batch number (i.e. to display a total of the previous day’s receipts)



例如现在current_batch = x. 3个Transaction， Timeorder T2 < T3< T1.  T2会插入 current_batch = x 的 receipt，但是T1会看不到这样一个Receipt。这个异常很奇特，T1作为只读事务参与到了整个异常中，如果没有T1。 T2 - T3 会按照顺序执行，符合序列化定义。



Snapshot Isolation (SI) is widely used, and many techniques have been developed to avoid anomalies:

- some workloads simply do not experience any anomalies; their behavior is serializable under snapshot isolation.
- potential conflict between two transactions is identified, explicit locking can be used to avoid it.
- the conflit can be materialized by creating a dummy row to represent the conflit, and forcing every transaction involved to  update that row.
- Constraint can be expressed to the DBMS (foreign key, uniqueness or exclusion constraint)





如何从理论的角度去解析这些异常，并尝试规避他们



**Snapshot Isolation Anomalies** (Theoritical)

Multiversion serialization history graph is introduced. Graph contains a node per transaction, and an edge between nodes per dependency. Three types of dependencies can create these edges:

- **wr-dependencies**: If T1 writes a version of an object, and T2 read that version, then T1 appears to have executed before T2.
- **ww-dependencies**: If T1 write a version of some object, and T2 replaces that version with the next version, then T1 appears to ahve executed before T2.
- **rw-antidependencies** * : if T1 writes a version of some object, and T2 reads the previos version of that object, then T1 appears to have executed after T2, because T2 did not see its update.  These dependencies are central to SSI, sometimes it also called **rw-conflict**.

wr and ww dependencies can be seen as one transaction is done and another is to begin. However rw-antidependencies occur between concurrent transactions: one must start while the other was active.

Therefore, it play an important role in SI anomalies.



According to above definetion, we can pic the example anomalies' history graph.

![historyGraph](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/SSI/historyGraph.png)

> **Theorem 1**
>
> *Every cycle in the serialization history graph contains a sequence of edges T1 ->(rw) T2 ->(rw) T3 where each edge is a rw-antidependency.*
>
> *Furthermore, T3 must be the first transaction in the cycle to commit.*





Thus, According Theorem1, somebody (SSI Papers) want to detect potential anomalies through checking the two adjacent rw-antidependency edge. They call it "dangerous structure". Althought it may have false positives since not every dangerous structure is part of a cycle (like above example), it dosen't have to track all history graph which is more efficient.



**SIREAD Lock**

SSI paper describe a method for identifying these dependencies by having transactions acquire locks in a special **SIREAD** mode on the data they read. These locks do not block conflicting writies. SIREAD locks must persist after a transaction commits, because conflicts can occur even after the reader has committed. This lock must be retained until all concurrent transactions commit.





#### Read-Only Transaction Optimizations

> **Theorem 2** : Every serialization anomaly contains a dangerous structure (two adjacent rw-antidependency), where if T1 is read-only, T3 must have committed before T1 took its snapshot.



if a dangerous structure is detected where T1 is read-only, it can be disregarded as a false positive unless T3 committed before T1’s snapshot. 

This result means that whether a read-only transaction can be a part of a dangerous structure depends only on when it takes its snapshot, not its commit time. 

If we can prove that a particular transaction will never be involved in a serialization anomaly, then that transaction can be run using standard snapshot isolation, without the need to track readsets for SSI.

. A read-only transaction T1 cannot have a rw-conflict pointing in, as it did not perform any writes. The only way it can be part of a dangerous structure, therefore, is if it has a conflict out to a concurrent read/write transaction T2, and T2 has a conflict out to a third transaction T3 that committed before T1’s snapshot. If no such T2 exists, then T1 will never cause a serialization failure. This depends only on the concurrent transactions, not on T1’s behavior; therefore, we describe it as a property of the snapshot:

- **Safe snapshots**: A read-only transaction T has a safe snapshot if no concurrent read/write transaction has committed with a rw-antidependency out to a transaction that committed before T’s snapshot, or has the possibility to do so.



**SAFE SNAPSHOT OPTIMIZATION IMPLEMENTATION**

when a READ ONLY transaction is started, PostgreSQL makes a list of concurrent transactions. The read-only transaction executes as normal, maintaining SIREAD locks and other SSI state, until those transactions commit. After they have committed, if the snapshot is deemed safe, the read-only transaction can drop its SIREAD locks, essentially becoming a REPEATABLE READ (snapshot isolation) transaction. An important special case is a snapshot taken when no read/write transactions are active; such a snapshot is immediately safe and a read-only transaction using it incurs no SSI overhead.



**Deferrable Transactions**

Read-only serializable transactions can be marked as deferrable with a new Keyword, *BEGIN TRANSACTION READ ONLY, DEFERRABLE*. Deferrable transactions always run on a safe snapshot, but may block before their first query.

The keypoint of Deferrable Transaction is to get a safe snapshot.





### Implementation

Unlike other DB , mysql (innodb) and BerkeleyDB,which provide strict Serializability (S2PL), PostgreSQL is completely based on MVCC.

Before SSI, there are two level of isolation

*Serializable* mode provide snapshot isolation : every command in a transaction sees the same snapshot of the database, and write locks prevent concurrent updates to same tuple.

*Read Committed* level, takes a snapshot before each query rather than using same snapshot during transaction.

In PostgresSQL 9.1, the *Serializable* level now uses SSI.



**Each tuple is tagged with the transaction ID of the transaction that created it (xmin), and, if it has been deleted or replaced with a new version, the transaction that did so (xmax).** The new tuple has a separate location in the heap, and may have separate index entries.2 Here, PostgreSQL differs from other MVCC implementations (e.g. Oracle’s) that update tuples in-place and keep a separate rollback log.



How to detect rw-conflict in PostgreSQL ?

**If the write happens first**, then the conflict can be inferred from the MVCC data, without using locks

Whenever a transaction reads a tuple, it performs a visibility check, inspecting the tuple’s xmin and xmax to determine whether the tuple is visible in the transaction’s snapshot.

If the tuple is not visible because the transaction that created it had not committed when the reader took its snapshot, that indicates a rw-conflict: the reader must appear before the writer in the serial order.

**We also need to handle the case where the read happens before the write.**

This cannot be done using MVCC data alone; it requires tracking read dependencies using SIREAD locks.

we developed a new SSI lock manager. The SSI lock manager stores only SIREAD locks. It does not support any other lock modes, and hence cannot block.



**Implementation of SSI Lock Manager**

Reads acquire SIREAD locks on all tuples they access, and index access methods acquire SIREAD locks on the “gaps” to detect phantoms. 

In particular, SIREAD locks must be kept up to date when concurrent transactions modify the schema with data-definition language (DDL) statements.

**Track Conflict**

this paper chose to keep a list of all rw-antidependencies in or out for each transaction. Keeping pointers to the other transaction involved in the rw-antidependency, rather than a simple flag.

It also implement the commit ordering optimization described in therorem1 and read-only transaction optimization. 

**Resolving Conflicts**

When dangerous structure is found, and the commit ordering conditions are satisfied, some transaction must be aborted to prevent a possible serializability violation.

**Safe retry**: if a transaction is aborted, immediately retrying the same transaction will not cause it to fail again with the same serialization failure.

The following rules are used to ensure safe retry: (for example two in above figure)

- Do not abort anything until T3 commits.
- Always choose to abort T2 if possible
- If both T2 and T3 have commited when the dangerous structure is detected, the only option is to abort T1. It is safe.

Rule 1 means that dangerous structures may not be resolved immediately when they are detected.





#### Memory Usage Mitigation

Four techniques are used to limit the memory usage of the SSI lock manager.

- Safe snapshots and deferrable transactions, which can reduce the impact of long-running read-only transaction.
- Granularity promotion: multiple fine-grained locks can be combined into a single coarse-grained lock to reduce space.
- Aggressive cleanup of committed transactions
- Summarization of committed transaction. If necessary, the state of multiple committed transactions can be consolidated into a more compact representation.



**Aggressive Cleanup**

a committed transaction's SIREAD locks are no longer necessary once all concurrent transactions have committed, as only concurrent transactions can be involved in a rw-antidependency.

Therefore, we clean up unnecessary locks when the oldest active transaction commits.

we record an additional piece of information in each transaction’s node: the commit sequence number of the earliest committed transaction to which it has a conflict out.

We use another optimization when the only remaining active transactions are read-only. In this case, the SIREAD locks of all committed transactions can be safely discarded.

**Summarizing Committed Transactions**

Our SSI implementation reserves storage for a fixed number of committed transactions. If more committed transactions need to be tracked, we summarize the state of previously committed transactions. 

It is usually sufficient to discover that a transaction has a conflict with some previously committed transaction, but not which one. 





### Conclusion

PostgreSQL 9.1 provide a new SSI-based serializable isolation level. This paper shows that this serializable mode provide performance similar to snapshot isolation and considerably outperforms strict two-phase locking on read-intensive workloads.



this implementation of SSI is the first in production use. 

It implement a new predicate lock manager that tracks the read dependencies of transactions, 

integrate SSI with existing PostgreSQL features, 

develop a transaction summarization tech to bound memory usage.

also introduce optimizations for read-only transactions