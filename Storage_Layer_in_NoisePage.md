# Storage Layer in NoisePage 

>this blog is about the analysis of noisepage's storage API and tutorial of perf in docker
>
>Author : Lingze

The overview of the whole layer.

![Storage](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/noisepage/StorageOverview.png)

NoisePage choose to store data by columns. `Data_Table` is a logical data structure, it organize the physical storage unit `Block`, every block's size is 1MB and provide simple `Insert`, `Delete`, `Update`, `Scan` API for upper transaction manager and execution engine. This blog will introduce the logic in `Data_table` in detail.

## How the noisePage organizes its block ?

`Data_Table` is a pre-defined table in database for user. Considering how user to create a new table, the first step is to define the type of every attribute in table, For example, Integer, char, VarChar(256).  Thus,  noisepage can calculate the size of every tuple (for variable length attribute, it will be disccussed in following section).  If the size of block is fixed,  the layout of this block can be determined. This information is stored in a read-only data structure named `BlockLayout`, If you check the source code of `BlockLayout`, the all data is arranged properly when you input am array of attribute size .

Block is divided  into two parts, header and data. Header stores some meta information, include static header and a RawBitMap. The RawBitMap is to record whether some tuples are existed, can help to allocate new tuple (flip 0 to 1 and insert data). Data parts contains a arrray of  MiniBlocks, these MiniBlocks can be located by the offset in header. Every MiniBlock store a series of data item in one column. The `null_bitmap` is used to record whether data is null. For example, Tuple_6's attribute_3 is null. We have to find the MiniBlock the attribute_3 correspond first and flip the 6th bit in null_bitmap to 0.

For the layout of noisepage, this is detailed paper [PAX](https://www.pdl.cmu.edu/PDL-FTP/Database/pax.pdf) to introduce.

## Storage Projection

There are two main structure for projection, `ProjectedRow` and `ProjectedColumn`. `ProjectedRow` is a sub-Tuple. `ProjectedColumn` is a serise of `ProjectedRow`. In `ProjectedColumn`, `RowView` is an inner class which provide access to the underlying logical projected rows with the same interface as a real `ProjectedRow`. In `Data_Table` , The `ProjectedRow` is like undo buffer and the `ProjectedColumns` is the result buffer of `Scan` Operation. The layout is shown in fig.  There is a generic template variable named `RowType` to represent `ProjectedRow` and `RowView`.

![Projection](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/noisepage/Projection.png)



## Table interface

Firstly, we have to get the subject for the design of noisepage. The MVCC of noisepage is Newer-to-Older and Delta-record, which means that tuple in block is always the latest version, txn should follow the delta-chain to materialize the tuple according to txn's timestamp. 

There are some public interface that `data_table` provide.  `Update` , `Insert`, `Delete`, `Scan`. I will introduce them separatly in code level.

![NoisePageScan](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/noisepage/NoisePageRead.png)

We can see that in Block, the tuple is always the newest. It reserved a column for version chain. The version chain's order is from newest-to-oldest. All Node (undo record) is stored in Transaction's buffer. (It means we can not casually free the transaction even it is committed. Only when consolidating the version chain, it can free some out-of-dated That's GC should do).

#### Scan

Firstly, I introduce `Scan`, read-only operation. Scan will invoke a sub-function `SelectIntoBuffer` to copy partial tuple into `ProjectedCol` following the slotIterator loop. The `SelectIntoBuffer` function's process is shown in fig.

Now we simply describe what `SelectIntoBuffer` dose. First, it will copy the newest version from Block in data_table into a Buffer. However, this version's tuple maybe not visiable for TxnC (shown in figure). It will follow the version chain to traverse and apply delta change to materialize the tuple. Firstly, it gets the undoRecord in TxnA and check the Timestamp that delta ts is older than TxnC's Ts ,then apply delta change.  Following the version chain, it gets the undoRecord in TxnB, but finds that the Ts of undoRecord in TxnB is smaller than TxnC's Tx, which means this delta change is not visiable for TxnC. Thus, it stop traversing and write back the out_buffer to the RowView in ProjectedCol.

In `Scan` interface, the `SelectIntoBuffer` function will be invoked, following the slot iteration.



#### Insert

`Insert`, `Update` and `Delete`,  are all write operation. For write operation, noisePage use strict and coarse-grained protocal. the order of version chain in noisepage is from newest to oldest, which means every write-operation have to change data in block. It use CAS(Compare-And-Swap)  to update new data, which can keep lock-free. 

Every Txn inserting new tuple in one block should set this block busy first. It means only one thread be able to try to insert into block at a time.

The `Insert` Operation 

- Find one block under this data_tuple has left tuple position
- try to use CAS to set this block busy. If failed, it means other txn is allocate new tuple in this block, back to step 1. If success, get the new tuple's slot.
- insert new tuple in the allocated tupleslot, update the version chain.



