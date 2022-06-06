# The design of Cohort Query Processing

### The Pattern of Cohort Execution

Refering to the Paper of [Cohona](http://www.vldb.org/pvldb/vol10/p1-ooi.pdf), we can break down a complete cohort query into five sub-query, shown in fig.  The target of query is to 

>Given the `launch` birth action and the activity table as show in Table (check raw paper), for players who play the `dwarf` role at their birth time, cohort those players based on their birth countries and report the total gold that country launch cohorts spent since they were born.

We call these five step 1). BirthSelection 2). BirthTupleGeneration 3). AgeGeneration 4). CohortAttributeGeneration 5). CohortGenration.

![QueryAnalysis](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/main/assets/Cohana/QueryAnalysis.png)

Actually,  we can compress step 1 and step 2 into one operation. The combination of step 1 and step 2 is called BirthSelection. As for step 4, it is not neccessary for the basic cohort query, it only add some additional attribution for a kind of cohort. For above example query, the target is to count the number of player for one country which is the cohort. From the example in paper [Cohort Analysis with ease](https://dl.acm.org/doi/10.1145/3183713.3193540). we can see that CohortAtributeGeneration is not the basic step for a cohort query.  Progress is shown in fig that step a,b,c generate data in axis-x and axis-y, step e generate lines of different color.

![CohortExample](https://raw.githubusercontent.com/Zrealshadow/Intro2DB/f3a0a4d582a9562174ff548bb824871a6face99c/assets/Cohana/CohortExample.png)

Thus we can summary the tree step of the basic cohort query, which is also the three operator in our implementation.

- BirthSelection
- AgeGeneration
- CohortGeneration

These three steps should be executed step by step in a CohortExecuteUnit.

### Pattern Design

(Following part is for cohort query processing only. We can consider this part as a component of the whole query processing module.  If we want to maintain a good open source project, we have to support base query process in the future)

There are some assumptions before we build the query execution part.

For one table, we allow append operation to add data. The data in every append operation is called a append block. Considering the application scenarios of cohort, we assume that *Every Append Block contains a bunch of behavior data item within a certain time window*.  For the layout of the Append Block, we also assume that *tuples are clustered in userId, and for one user, its tuple is sorted in time serise*. These are two constrains we set for input data.

First we create a class `CohortOperator` to combine above three sub-process and also CohortAttributeGeneration step. (This step can be null). For three sub-process, we can create classes separately. There is no need to allow them to implement same interface, since these three classes are all needed in Cohort Processing. However, every class have to implement `init` and `process` two functions. `init` load filter from query.

Simply introduce the logic of every sub-process.

The logic of BirthSelection's `process`

1. Iterate append block, check whether the time feature's block obeys the condistion of filters.
2. For one block, iterate the Cublet, check the meta info whether possible birth tuple is existed.
3. If there are possible birth tuples, check the data chunk, for eligible tuple, we record the userId (String) and BirthTime in a HashMap. 
4. `process` return these HashMap

The logic of AgeGeneration's `process`

1. The Iterate and necessary check pattern is like above BirthSelection's process
2. In data chunk, for eligible tuple, we record their unique identifier among whole table in a ArrayList (CubletId-offsetInRLE).
3. `process` return the ArrayList which contain accessible tuple. **These ArrayList is the scope the rest operator applied on.** Each element of this ArrayList is a Array contain accessible tuple. The index of the ArrayList is age in cohort.

The logic of CohortGeneration's `process`

1) For every element in Arraylist returned by AgeGeneration's `process`, We maintain  HashMap \<Cohort - SelectedValueList\>. 
2) Iterate these Accessible Tuple, store these value and apply certain analysis function.

In order to reduce the usage of memory, we can implement the `CohortOperator` as an iterator container. Everytime `step` is called, it return the i-th age's cohort data (HashMap).



Above content is the simply imagination of modular cohort query's execution processing. I think it can solve the continuous problem. Additionally, apart from cohort query, we should reconstruct the filter class in the next few days. Certainly, there are many other aspects to consider in code level, such as the memory usage and the query optimization, which are left in the future work.









