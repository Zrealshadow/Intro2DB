# Bw-Tree's Base and Extension (1)

>Source Paper:
>
>- [The Bw-Tree: A B-tree for New Hardware](https://15721.courses.cs.cmu.edu/spring2020/papers/06-oltpindexes1/bwtree-icde2013.pdf)
>- [Building a Bw-Tree Takes More Than just Buzz Words](https://15721.courses.cs.cmu.edu/spring2020/papers/06-oltpindexes1/mod342-wangA.pdf)





### Abstract

This paper introduces a new ARS (atomic record stores) that provide very high performance, named Bw-tree. And its associated storage manager particularly appropriate for the new hardware environment.

The advantage of Bw-tree

- latch-free technique, all operation is CAS without setting latched.
- this structure carefully avoids cache line invalidations, hence leading to substantially better caching performance.



### Motivation

We live in a high peak performance multi-core world. We need to get better at exploiting a large number of cores by addressing at least two important aspects:

- As the concurrency increases, latches are more likely to block, limiting scalability.
- high CPU cache hit ratio is important. Updating memory in place results in cache invalidations, so how and when updates are done needs great care.

For first challenge, the Bw-treee is latch-free. For second issue, the Bw-tree performs "delta" updates that **avoid updating a page in place.**





### Bw-Tree Architecture

The Bw-tree **atomic record store** (not just a index structure) layout 

![bwtree-layout](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/bwtree/bwtree-layout.png)

**Bw-tree Layer**

The access method layer is at the top. It interacts with middle *Cache Layer*. It organize the pages in memory as a tree and offer the acccess interface (search and update) of this tree.

**Cache Layer**

In Bw-tree Layer, all page are identifiered by a logic PID. Cache Layer transfer the PID to the page in memory. It maintain a mappting table, mapping the logic page to phyisial page. It also manage the pages in and out in memory.

**Flash Layer**

Manage write to flash storage. Not like traditional storage engine like b+tree, which writes in place and in a fixed page size, the page size in Bw-tree is elastic.  It transfer the random write to  serializable write, improving the performance of write.





#### Mapping Table 

mapping table maps logic pages to physical pages. Logical pages are identified by a logical "Page identifier" or PID. The mapping table translates a PID into a **flash offset**,  the address of a page on stable storage or a **memory pointer**, the address of the page in memory. 

This "relocation" tolerance directly enables both delta updating of the node in main memory and log structuring of our stable storage.  *for this disadvantage, We can compare with B+tree storage engine*, Bw-tree nodes are thus logical and do not occupy fixed physical location. 

Permitting page size to be elastic, meaning that we can split when convenient as size constrains do not impose a splitting requriement.



#### Delta Updating

Page state changes are down by creating a delta record (describing the change) and prepending it to a existing page.

It install the new memory address of the delta record into the page's physical address slot in the mapping table using CAS instruction.

This delta updating is one of the most innovation of Bw-tree, which simultaneously enables latch-free access in Bw-tree and preserves processor data caches by avoiding update-in-place.



#### Log Structured Store (LSS)

Like usual LSS, pages are writtten sequentially in a large batch, reduce the I/Os required. When flush a page, the LSS only need flush the deltas that represent the changes made to the page since its previous flush, which dramatically reduce how much data is written, increasing the number of pages that fit in flush buffer. (Traditional Bw-tree will write all page although there is a tiny modification, -- write amplification).

The LSS cleans prior parts of flash representing the old parts of its log storage. It also makes pages and their deltas contiguous for imrpoved access performance.

However, in this paper, this section is not showed in detail.





### In memory Latch Free Pages

All structure is similar with B+ tree. Two distinct features make the Bw-tree page design unique. 1). use PID instead of physical address as value in branch node. 2) Pages are elastic, there is no hard limit on how large a page may grow.

How to update. See pic

![bwtree-update](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/bwtree/bwtree-update.png)

First create a new delta record D that points to the page's current address P.  obtain P from the page's entry in the mapping table. The memory address of the delta record will serve as the new memory address for this page. To install new page state in mapping table, this paper use CAS instruction to replace the current address P with the address of D.

Since updates are atomic, only one updater can "win" if there are competing threads trying install a delta against the same "old" state in the mapping table. A thread must retry its update if it fails in CAS.

How to seach:

Traverse the delta chain. The search stops at the first occurrence of the search key in the chain. if the delta represents a delete, the search fails. If the delta chain dose not contain key, the search performs a binary seach on base page.



How to consolidate page ?

To prevent delta chains growing too long, Bw-tree occassionally perform page consolidation that creates a new "re-organized" base page containing all the entries from the original base page as modified by the updates from the delta chain.

When consolidating, the thread creates a new base page(a new block of memory) and it then populates the base page with a sorted vector containing the most recent version of record from either the delta chain or old base page. The install the new address of the consolidated page in the mapping table. **If the CAS fails, this consolidation will not retry. as a subsequent thread will eventually perform a successful consolidation**.



How is the Garbage Collection work ?

(Actually, the description of Grabage collection in this paper is vague)

A latch-free environment dosen't permit exclusive access to shared data structures, which means that there are multi-thread will access one page. If one thread consolidate the old page and create a new base page, the old page can not be reclaimed immediately. Some other thread may read the content in this old page. The Garbage Collection should guarantee that no other thread is reading this page when reclaiming it.

An epoch mechanism is introduced, which is not showed in detail in base Bwtree paper but it is explained and improved in [Extension paper](https://15721.courses.cs.cmu.edu/spring2020/papers/06-oltpindexes1/mod342-wangA.pdf).



### Bw-tree Structure Modification

As we all know, the structure modification in B+ tree is the most difficult and complicated operation. How to integrate these structure modification (Merge and split) in previos delta record and keep latch-free ?

![bwtree-SMO-split](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/bwtree/bwtree-SMO-split.png)

**Node Split**

Splits are triggered by an accessor thread that notices a page size has grown beyond a system threshold. This Structure Modification Operation (SMO) is divided into 2 phases. shown in pic.

1.  Child Split, the Bw-tree requests allocation for a new node Q in mapping table. Then find the appropriate separator key from P that provides a balanced split and proceed to create a new base node Q. Install the physical address of Q in the mapping table. *No need to use CAS, Q is only visible to this thread*. Then prepend a *Split delta record* to P, which contain two pieces of info 1) the separator key K . 2) a logical side point to the new sibling (PID). ----- Now page Q is valid even without the index in parent node O. thread can search through split delta record in delta chain of Page P.
2. Parent update, prepend an index term delta record to the parent of P and Q. Now we can directly search page Q from parent O instead of traverse delta chain of Page P.

Posting deltas decreases latency when installing splits, relative to creating and installing completely new base pages for Page P and Q.

( Actually, there is a problem that before c) step in pic, there is a updatein Page Q. The delta record will be prepended in Page P's delta chain. After c) step, when index entry delta record added into parent Page P, these modication will miss. The following section will introduce a solution for this problem.)



**Node Merge**

![bwtree-SMO-merge](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/bwtree/bwtree-SMO-merge.png)**

- Marking for delete. Prepend a remove delta record on Page R. Any thread encountering a remove node delta in R needs to read or update the contents of R previously contained in R by going to the left sibling.
- Merging Children. The left sibling L of R is updated with a node merge delta, which physically points the content of R.
- Parent Update. The Parent node P of R is now updated by deleting its index term associated with R.



### Serializing Structure Modifications and Updates.

Because that the split and merge all are serise of atomic operation, it is neccessary to consider if other atomic operations occur in these SMOs.

How to solve these.

For conflict between update and SMO, when a thread want to update a Page L, but find a intermediate state of SMO.  It will not continue its update until this SMO is committed completely.

In short, For a same object, the new update is not allowed to installed between a SMOs.

(But I don't know how to implement it)



For conflict between SMO and SMO, it use a stack of SMOs to keep all SMOs atomic operation are serialized. 

But there is no detailed description on the implementation.





---

**the above text is all about Bw-tree Layer** 



Cache Management

This paper design a special Cache management and a special LSS to cooperate to implement a transaction mechanism.



First, we have to seperate the transaction component from Data Component.

LSN, Log Sequence Number which is used to identify the delta record, is generated by Transaction Component.

How these two component to cooperate ?

When TC write record into disk, it needs to update their ESL (End of stable Log). ESL means any record whose LSN smaller than ESL have been written into disk. For example, Txn_A contains LSN1, LSN2, LSN3. When Txn_A is committed, it means that all three records are durable and written to disk. ESL will be update to LSN3. TC will send ESL to DC periodically and remind DC that any delta records which LSN is largger than ESL are not visitable (Uncommit). DC will not persist log whose LSN are largger than ESL.



why DC have to persist pages ? just like checkpoint in recovery.  It can help TC to push Redo-Scan-Start-Point.(RSSP)

TC send a request to DC to push RSSP. When DC got these, it will persist all records whose LSN are smaller than RSSP. When TC got the response, TC is allowed to delete those record which are been persist in DC, because TC no need to send these records while recovery.



**How to flush Pages to the LSS**

**Flush detla record**

When Flush a page into disk successfully, it will add a *fulsh delta record* in the delta chain, keeping track of which delta records were flushed into disk. Now in next persistance, only flushing append delta is enough.

**Page Marshalling.**

When cache management want to flush a page into disk, page state will be **captured**, which indicate that this page is in flushing, other thread can not do any update in it. 



**Incremental flushing**

when flush a page, cache manager only decoding records whose LSN between MAX LSN record in last flush and ESL.

(Actually, if these page is been consolidated, the whole page has to be flushed again.However, even for this, compared to B+tree that only one modification, the whole page should be flushed, this optimization is good idea. )



### 总结





