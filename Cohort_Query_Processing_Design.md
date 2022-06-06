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

These three step should be executed step by step in a CohortExecuteUnit.

### Pattern Design

There are some assumptions before we build the query execution part.

For one table, we allow append operation to add data. The data in every append operation is called a append block. Considering the application scenarios of cohort, we assume that *Every Append Block contains a bunch of behavior data item within a certain time window*.  

