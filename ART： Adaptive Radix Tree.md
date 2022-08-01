# ART： Adaptive Radix Tree

## Motivation

Three kinds of indexs : 1. B+ Tree variants 2. Hash table 3. Trie, radix tree, prefix tree.

B+-tree variants requires more expensive update operation.Another drawback is the result of comparisons in B+-Tree variants can not be predicted easily. The long pipelines of modern CPUs stall, which causes additional latencies after every second comparison (on average).

Hash Table are less commonly used as database indexs. The first reason is that hash tables scatter the keys randomly, only support point queries.  Another problem is most hash tables do not handle growth gracefully, requiring reorganize upon overflow with O(n) complexity.

## Intro

ART is a kind of trie, which is a fast and space-efficient in-memory indexing structure. While most radix trees require to trade off tree height versus space efficiency by setting a globally valid fanout parameter, ART adapst the representation of every individual node. In short, every innr node is different with each other. By adapting each inner node locally, it optimizes global space utilization and access efficiency at the same time. Nodes are represented using a samll number of efficient and compact data structures. Two additional techniques, path compression and lazy expansion, allow ART to efficiently index long keys by collapsing nodes and thereby decreasing the tree height.



**Adaptive Node**

The key idea that achieves both space and time efficiency is to adaptively use different node sizes with the same, relatively large span, but with different fanout. You can see the difference between ART and other radix tree.

![ART](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/ART/ART.png)

There four kinds of inner nodes. node header

- Node4, the smallest node type. four `unsigned char ` key and four child pointer. 
- Node16, 16 `unsigned char` key and 16 child pointer
- Node48, a little different. 256 `unsigned char ` and 48 child pointers. In put key is a `unsigned char`, convert it to a value and take it as a index in the 256 array. and every element in 256 array saves the index of child pointers.
- Node256, 256 child pointers. The value of key byte will be converted directly to the index of child pointer.

![ART](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/ART/ART_node.png)

Space consumption for these four kinds of inner node

![ART](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/ART/ARTNodeConsumption.png)

**Collapsing Inner Nodes**

For some long key, we don't have to build a inner node for every part of it. The following figure shows the techniques

Lazy expansion, inner nodes are only created if they are erquired to distinguish at least two leaf nodes.

Path Compression, removes all inner nodes that have only a single child. Two approaches to deal with it:

- Pessimistic: at each inner node, a variable length partial key vector is stored
- Optimisitic: only the count of preceding one-way nodes is stored. when lookup arrives at a leaf. Its key must be compared to the search key to ensure no wrong turn was taken.

![ART](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/ART/CollapsingInnerNode.png)



**Constructing Binary-Comparable Keys**

Keys are divided into several bytes and then we can take the ART as index.  We should keep that the order compared by sets of bytes is same as the order of original key.

*Acutally, it's not necessary to consider this point when implement the ART index data structure*



### Exp

The following fig shows the performanc of ART. It works better than  most other index.

![ART](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/ART/Exp.png)

The following fig shows the effect of lazy expansion and path compress techniques. It can significantly reduce the height of the index.

![ART](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/ART/exp_compress.png)

 