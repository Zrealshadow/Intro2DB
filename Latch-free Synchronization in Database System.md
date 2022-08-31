# Latch-free Synchronization in Database System

>Lingze
>
>[Paper](https://15721.courses.cs.cmu.edu/spring2020/papers/06-oltpindexes1/faleiro-cidr17.pdf) Rethink the Latch-free technology's application in different sceneries in multi-core hardware.
>
>Conclusion: Latch-free algorithms usually outperform their latch-based counterparts when the number of OS contexts exceeds the number of cores in the systems.

（这篇note用中文写QAQ）



### 对Latch-free algorthims的定义

Latch-free algorithms主要是相对于Latch-based algorithm来说的。

最主要的区别是，no thread is blocked due to the delay or failure of other threads. 对比latch-based algorithm. latch-based algorithm通过互斥锁(mutual exclusion) 的概念保证在并行条件下只有一个thread可以访问一段共享数据结构。当一个thread拥有这个latch，其他的thread就会被阻止。



### 扩展性（Scalability）

在这一部分，paper简单的介绍了几种Latch-base algorithm和通常的latch-free algorithm。

**最简单latch通过test-and-set去实现**， 一个test-and-set会无条件的将内存设定为指定的值，然后返回原来的值。如果test-and-set(1)操作后返回0，代表thread给成功上锁。返回1代表有其他thread持有锁。

上述实现的问题是这样不停的retry 去写内存可能会给latch value的内存管理加压，而且增加了NUMA node 的冲突。（Performing test-and-sets in a tight loop puts pressure on the memory controller associated with the latch word and increases traffic across NUMA nodes.）

**针对上述问题提出了(test-and-test-and-set | TATAS) latch.** 

如果该值0，成功test-and-set(1) 将值置为1，上锁成功。该值为1，说明其他thread拥有该锁，没有test-and-set(1)操作，上锁失败。避免了无谓的写操作。但是还是会retry去做 test-and-test-and-set 操作。

TATAS latch解决了无条件写的问题，但是这里他提到了一个非常细微的点就是，TATAS 非常看临界区段（critical section）我这里认为是thread拿到latch之后所工作的这段时间。如果临界区段非常的小，那么TATAS的性能主要在锁的转移上（transient behavior）。在这一段中，T0 释放了锁，T1-Tn都会执行TATAS去获得锁，虽然T1执行成功了，但是不会阻止其他的Thread 去执行这个TATAS过程。这里造成了大量的test-and-set请求（It cause transient flood of test-and-sets requests）。 只有当这些所有的请求结束之后，系统才会处于静默阶段（quiesce）.

TATAS的问题也非常明显，所有thread盯着（spin on）一块全局内存（global memory location）, 这会造成扩展瓶颈 (scalability bottlenecks).

**Backoff mechanisms**， 官方的解释是在atomic操作和读之间塞了很多noop instruction。在实验中Pthread实现了这个机制。直接让失败的TATAS sleep一段时间。

**Scalable latching data-structures**, threads盯着本地内存或者一个core-local数据结构。在实验中是MCS latch. 每一个thread有一个相关的队列节点，这个对俩节点会被用atomic操作加入到一个队列中，这个队列的第一个节点对应的thread拥有这个latch。When the latch holder completes its critical section, it sets the
flag in the next queue node. The corresponding thread then notices
this change and begins executing the critical section.

**Free-latch algorithm**, 这个就是使用cmp-and-swap常规操作。

文中提到了，Latch-based 和 Free-based 有点像concurrency-control中的Pessimistic and Optimisitic. 但是这篇为文章意在指出这两种algorithm的相同点。



### Scheduling requests

在这一部分，文章提到了这样一个场景，如果OS 上下文（thread）多于CPU的核的个数 (If the number of OS contexts exceeds the number of available CPU cores). 就可能出现一个CPU被两个thread复用的情况。操作系统会自动给定时间片给每一个上下文。当一个时间片过期另外一个就会抢占CPU。（The Scheduling of OS contexts is handled by the operating system, which assigns a fixed time slice to each context, and preempts a context when its slice expire.）在后面的实验中，可以看到在这种情况下，上述所说的抢占CPU会极大的影响性能。例如，如果有两个thread，T1拥有latch，T2等待latch。但是现在CPU core 被分配到T2使用，那么这样整个处理的性能就会被拖延。这种时间片的划分不再用户空间内，属于kernel space。

从理论上来说，如果thread的个数多于CPU核的个数，free-latch algoirthm is better than latch-based algoirthm. 因为上述情况中，free-latch algorithm 不会因为CPU core的分配而受到太大的影响。因为每个thread不会因为其他Thread的延迟而延迟。

### Memory management

在内存管理上Free-Latch algorithm也要比Latch-based algorithm要复杂。在之前BW-Tree的论文上，就可以看到，主要考虑到两点：

- 复制带来的过载
- 垃圾回收
- 内存的复用

对于复制带来的过载（Copying overhead）是因为，Free-Latch algorithm不支持就地更新（in-place update）.， 通常会复制一份或者部分需要更改的数据结构，在副本进行更改后，通过atomic 操作对其进行替换。这样如果数据结构非常的复杂，那么在Free-latch algorithm中，复制的成本和代价就会非常高。文中还提到了，操作数据结构的副本，也会导致较较差的缓存利用率，对比就地修改来说。

对于垃圾回收（Garbage collection）, 因为在Free-latch algorithm中是没有读锁的，可能存在删除一个数据结构对象，但是该对象正在被其他thread读取。因此删除操作不能直接删除，只有等该数据结构对象的所有引用都消除了，这个对象才能被删除。在BW-tree中，它使用的垃圾回收就是epoch-based reclamation.

对于内存复用（Memory re-use）, 文中提到了一个ABA问题，这里不加赘述。

文中提到，垃圾回收机制是为了让内存不要过早释放，ABA防止策略是为了让内存不要太早复用。





### 案例分析，数据库索引 B+tree

关于索引，我有很多note和learning  material，这里简略介绍。

首先就是对于B+tree，最基础的就是 Latch Coupling。这种从up-to-bottom的加锁方式有两点比较不太行， 第一个肯定是随着Tree的增大，对于上层节点的更改会越来越少，让上层节点持有锁，是一个明显的performance Bottlenettle。第二个在multi-core的情况下，虽然ReadLatch不会阻碍其他CPU core上thread的进行，但是Read Latch会有那种Count计数等变量，RL持续变动会使另 CPU cache consistancy对性能有很大的影响。

因此BlinkTree被提出，这是一种从下至上的加锁形式，只加写锁，对于读使用另外的机制保持一致性。在BLinkTree基础上，OLFIT使用Timestamp去保证读操作可以得到一致的结果。这种使用Timestamp去代替Latch的也可以被认为是一种Free-Latch algorithm.

BW-tree就不说了， 总结来说就是增加可扩展性的宗旨就是-- 避免频繁的更新共享内存结构。

both algorithms( free-latch algorithm and latch-based algorithm) are scalable because requests avoid updating frequently accessed shared memory.



### 实验

![LowContention](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/FreeLatchAnalysis/LowContention.png)

在LowContention情况下, 这里记录以下可以看到MCS在80thread之后有一个明显的Collapse。这是因为他的Latch的获取是一个等待队列， CPU preemption会delay整个队列的进展。 同样对于Latch-based 算法CPU preemption都导致会有这样的性能削减。但Free-based算法就不是这样。



![MediumContention](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/FreeLatchAnalysis/MediumContention.png)

对于Medium Contention 的情况，他认为Latch-free, Pthread, TATAS的下降都是因为，由于上文说的critical section非常小，然后transient time主导了这个性能。因为不同于MCS， Latch-free, Pthread, TATAS在Latch转换时，其他所有需要Latch的Thread都会并发争抢。如果这样一个单一的争抢过程的耗时比Critical section小，那么会导致拥有Latch的Thread释放了Latch，但是其他的Thread上一次的争抢过程还没结束。

但是个人认为这个是为了做实验而做实验，TMD，那有cirtical section这么小的应用场景。

该文还做了一个Queue的实验，也是证明了Latch-free Algorithm不比 Latch-Based Algorithm好很多。

该文也给了一个实验的总结，**在硬件层面上，唯一影响同步机制可扩展性的就是这些算法能否避免重复的读写一项特定的内存空间。**



### 结论

只有在Thread多于CPU core数量的情况下，Free-Latch算法效果会好于Latch-based算法。因此对于Latch-free因该进行谨慎的追捧。









