# BitCoin: A Peer-to-Peer Electronic Cash System

>Author : Lingze
>
>Note some keypoints about this paper from distributed system aspect And some basic concept to help me understand bitcoin.
>
>[Paper](https://pdos.csail.mit.edu/6.824/papers/bitcoin.pdf)
>
>[lecture note](http://nil.csail.mit.edu/6.824/2021/notes/l-bitcoin.txt)

The target of this paper:

>What is needed is an electronic payment system based on cryptographic proof instead of trust allowing any two willing parties to transact directly with each other without the need for a trusted third party.

These are challenges for these solution

- outright forgery
- double spending
- theft



**Some basic concept:**

What's the transaction

*It's an electronic coin as a chain of digital signatures*

shown in figure

![](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/BitCoin/Transaction.png)

contains

- pub(user1) : public key of new owner
- hash(prev) : hash of this coin's previous transaction record
- sig(user2) : signature over transaction by previous owner's private key

```
transaction example:
  Y owns a coin, previously given to it by X:
    T6: pub(X), ...
    T7: pub(Y), hash(T6), sig(X)
  Y buys a hamburger from Z and pays with this coin
    Z sends public key to Y
    Y creates a new transaction and signs it
    T8: pub(Z), hash(T7), sig(Y)
  Y sends transaction record to Z
  Z verifies:
    T8's sig(Y) corresponds to T7's pub(Y)
  Z gives hamburger to Y
```

Key point : **only the transactions exist, not the coin themselves. Z's "balance" is set of unspend transactions for which Z knows private key the "identity"  of a coin is the hash of its most recent xaction.**



Now let's us talk about the challenge of **Double spending**

the case is shown in following:

```
 Y creates two transactions for same coin: Y->Z, Y->Q
    both with hash(T7)
  Y shows different transactions to Z and Q
  both transactions look good, including signatures and hash
  now both Z and Q will give hamburgers to Y
```

now, if Z and Q didn't know complete set and order of transactions, double spending will become possible.

In real world, Z and Q will request the thrid-party like Bank to check the validation of transaction. In bitcoin paper, the author want to decentralize the system and leave the thrid-party institute behind.

So, if all transactions are published and everyone sees the same log, Z and Q will reject the transaction of double-spending.  This is where blockchain comes in.

The BitCoin block chain:

The goal is the agreement on transaction log to prevent double-spending. The block chain contains transactions on all coins.

Many peers -

- each with complete copy of the whole chain
- each with TCP connections to a few other peers 

the structure of each block :

- hash (prevblock)
- set of transactions
- "nonce", the puzzle for next block
- current time (wall clock timestamp

And the most importance is that the longest chain of BitCoin is valid, we call it "Nakamoto consensus". And the transaction in BitCoin blockchain is append-only and published. 



how to keep the valid chain is awalys the longest. The mechenism should encourage people to create new block.

How to create a new block. this is "mining" via "proof- of - work". The only thing we have to know is that it would likely take one CPU months to create one block. and when you create a new block, this mechanism will reward you with some BitCoin. 

Proof - of - work solves the Sybil problem.  

All peers will mine blocks based on the longest chain. If someone to modify the Transaction in block chain, it have to make up another longest chain, then the fake transaction will be valid. Because of Proof-of-work, the cost of fake chain is very high.  Considering the incentive mechenism, if the attacker want to pay the cost to assemble a fake chain, he would have to choose between using it to defraud people by stealing back his payments, or using it to generate new coins. (smart, LGTM)

















