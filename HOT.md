# HOT: A Height Optimized Trie Index for Main-Memory Database Systems

>Author : Lingze
>
>We mainly focus on the storage layout of HOT Index

### **Definetion**

Height Optimized Trie (HOT), a general-purpose index structure for main-memory database systems. HOT is a balanced design that efficiently supports all operations relevant for an index structure, but is particularly optimized for space efficiency and lookup performance.

### **Intro**

Height Optimized Trie (HOT). HOT combines multiple nodes of binary Patricia trie into compound nodes haveing a maximum node fanout of a predefined value k such that the height of the resulting structure is ptimized.

Thus, each node uses a custom span suitable to represent the discriminative bits of the combined nodes.

**A crucial property of HOT is that every compound node represents a binary Patricia trie with a fanout of up to k.** 

>简单来说，HOT的每一个Node都是一个利用Patricia优化过的Binary Tree. 每一个BiNode都代表着一个Bit置0，置1。相比于ART这些index最小的Partial Key 是一个Byte，HOT的内存适应粒度更细，他可以精确到Bit，因此更加的节省内存。实验也验证了这一猜想。另外为了防止每一个Node中二叉树过高，在创建Hot索引之前定义了统一的每个Node中树高的限制。
>
>图1示了一个HOT Index 的 Node中二叉树
>
>HOT Index 就是由这一个个Node由Pointer连接。
>
>图2展示了一个HOT的结构



![NodeInnerOverview](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTNodeOverview.png)

![HOTIndex](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTIndexOverview.png)

Because we have to keep the max height of each node in HOT less than K.  When comes to Insertion, if the BiNode mismatched, the mismatched BiNode has to split and expand. We call this overflow. We have two cases to solve them, ***intermediate node creations*** and ***parent pull up***.

Skip the detail of this section. The figure shows different cases when inserting a new Key. We focus on the storage layout of HOT index.

![NodeInsertion](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTInsertion.png)

### Node implementation

The key idea behind our node layout is to linearize a k-constrained trie to a acompact bit string that can be searched in parallel using SIMD instructions.

To achieve this, we store the discriminative bits of each key consecutively, shown in Figure a. By storing the partial keys consecutively, we can search all keys in parallel using SIMD operations instead of traversing the corresponding trie.

what's the discriminative bits?  It's the bit which can decide the path when trace the key. In figure a, for V3 key, we can see the 3th, 6th, 9th bit can judge the trace path. for V5 key, the 3th, 4th, bit can judge the trace path. So the bit position for this node is a set contains all keys' discriminative bits' position.

Thearetically, the partial keys which store the corresponding bit positions can help to match the whole key. 

In order to improve insertion performance, we actually use as slightly improved version of partial keys called *sparse partial key*.

What's the *sparse partial key* ? Actually, Not every bit position is discriminative for certain key. For example, the figure shows that the bit position for node is {3,4,6,8,9}. But for V4 key, the discriminative bit is {3,6,9}. In *sparse partial key*, we directly set non-discriminative bit to zero. Create a *sparse partial key* is more efficient than *dense partial key*.

![NodeNodeimplementation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTNodeImplementation.png)



Shown figure 5, we represent the bitpositions as a bit mask. To speed up key extraction, we therefore utilize the *PEXT* instruction from the BMI2 instruction set, which extracts bits specified by a bit mask from an integer.  This layout can be used whenever the smallest and largest bit positions are close to each other.



illustrated in figure 5, HOT also proposes a multi-mask layout, which breaks up the bit positions into multiple 8-bit masks. We can consider that these multi-mask can break 8 Bytes into a series of scope.

Uses 0 or 1 to set which is the left or right boundary.

![NodeNodeimplementation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTNodeLayout.png)

HOT provides different Storage layout.



### Synchronization Protocal

It is perfect fit for the Read-Optimized Write Exclusion (ROWEX).

The detailed of this section will be filled in the future.



### Expriment.

Figure 8 shows that in the different type of key and different operations, the performance of HOT is better than other baseline, like BTree, Masstree and ART. 

I doubt this result. Since the parallel operations in HOT is more complicated than ART or other tries index.

 ![NodeNodeimplementation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTExpThroughout.png)



Figure 9 shows the memory using for HOT. It can be seen that HOT has a significant effect in saving memory.

 ![NodeNodeimplementation](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/HOT/HOTExpMemory.png)





