# Staring into the Abyss: An Evaluation of Concurrency Control with One thousand Cores.

>[Link](https://15721.courses.cs.cmu.edu/spring2020/papers/02-inmemory/p209-yu.pdf)
>
>Note : many - core chips is different from many-node

This paper perfomred an evaluation of concurrency control for on-line trandsactionn processinng workloads on **many-core chips**. It identify fundamental bottlenecks that are independent of the particular database implementation and argue that even state-of-the-art DBMSs suffer from these limitation. It also give some possible solution for to overcome these shortcoming of these concurrency contral in many-core chips.



### Concurrency Control Schemes

![ConcurrencyControlSchemes](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Abyss/ConcurrencyControlSchemes.png)

2PL-DL : when deadlock is found, the system choose a transaction to abort.

2PL-NO_WAIT : when deadlock is found, transaction start to abort immediately.

2PL-WAIT_DEI: when deadlock is found, the younger transaction is forced to abort





### General Optimizations

##### Memory Allocation:

Customed `malloc` can automatically resize the memory pool based on the workload, which amortize the cost for each allocation when large chunks of contiguous memory is frequently allocated.

##### Lock Table:

Instead of having a centralized lock table or timestamp manager, it implements data structure with laches. In memory overhead, this meta-data (latches) is negligible for large tuples.





#### Optimization for 2PL

**Deadlock Detection** (Confusing)

Partition the data structure across cores and make the deadlock detector lock-free. It means every thread hold a own wait-for graph. In the deadlock detection process, a thread searches for cycles in a partial waits-for graph constructed by only reading the queues of related threads without locking the queses.



##### Lock Thrashing:

It designs transactions acuire locks in primary key order. Although this approach is not practical for all workloads, it removes the need for deadlock detection and helps to show the effects of thrashing.

![LockThrashing](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Abyss/LockThrashing.png)

From Figure 4, we can observe that with the increase of contention,  thrashing start to occur, the throughput decreases.



##### Waiting - Aborting

Add a timeout-threshold in DBMS that causes the sytsem to abort and restart any transaction that has been waiting for a lock for an amount of time greater than the threshold. Set timeout-threshold 100us.



#### Optimization for Timestamp Ordering.

##### Timestamp Allocation:

these alternatives for TS allocation:

- Mutex in allocation critical section
- atomic addition operation
- atomic addition with batching
- CPU clocks
- hadware counters

![TSbenchmark](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Abyss/TSbenchmark.png)

From the above figure, we observe that Clock and HWCounter work well in both No-Contention and Medium Contention, but there is no existed CPU support these techniques. Atomic operation is a good choice when chip has many cores, although it's performance is not good in no-Contention situation, but it's efficient as CPU clock and HWCounter when there is much contention.





### Experiment Analysis

This section is to conduct different kinds of experiment and analysize the effect of these concorrency control in different situation.

Check these section in [raw paper](https://15721.courses.cs.cmu.edu/spring2020/papers/02-inmemory/p209-yu.pdf)





### DBMS Bottlenecks.

Several bottlenecks to scala to many-core chips (unsolved)

- lock thrashing
- preemptive aborts
- deadlocks
- timestamp allocation
- memory-to-memory copy

And this figure shows the summary of the bottlenecks for each concurrency control scheme

![Summary](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Abyss/Summary.png)



### Conclusion

Results show that none of the algoritms are able to get good performance at such high core count in all situation.

For lower core configuration,  2PL-based algorithm are good at handlinng short transactions with low contention that are common in key-value workloads. Whereas T/O-based algorithms are good at handling higher contention with longer transactions that are more common in complex OLTP workloads.









