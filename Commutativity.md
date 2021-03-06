# Commutativity
\label{sec:commutativity}

- *discuss formal representations of commutativity from \cite{Herlihy:PPoPP08} and \cite{Kulkarni:PLDI11}, tradeoff between maximum concurrency and cost (runtime and implementation) of tracking it* 


Commutativity is well known, especially in distributed systems, for enabling important optimizations. Since the 80s, commutativity has been exploited by database systems designers \cite{Weihl:1988,Fekete:90}, within the safe confines of relational models, where knowledge of query plans and complete control of the data structures allows systems to determine when transactions may conflict. Recently, commutativity has seen a resurgence in systems without a predefined data model, such as NoSQL databases and transactional memory.

In the realm of eventual consistency, commutativity has been leveraged for convergence guarantees.
RedBlue consistency allows executing commutative ("blue") operations locally, knowing they will eventually converge. Similarly, conflict-free replicated data types (CRDTs) \cite{Shapiro:SSS11} define commutative merge functions for all operations to ensure that replicas will converge.

Lynx \cite{Zhang:SOSP13} uses knowledge of some commutative operations to make tracking serializability cheaper in chained transactions. Doppel \cite{Narula:OSDI14} added several explicitly commutative operations on records which they exploited to better handle common high-contention situations such as counters and "top-k lists" in the context of a single node multicore database. Finally, HyFlow \cite{Kim:EuroPar13}, a distributed transactional memory framework, reorders commutative operations on specific data types to execute before others to allow them to operate concurrently on a single version of a record.

## Commutativity Specifications

Though *commutativity* is often discussed in terms of an operation commuting with all other operations, it is actually more nuanced. If a pair of operations commute, then executing them in either order will produce the same result. Using the definitions from \cite{Kulkarni:PLDI11}, whether or not a pair of method invocations commute is a property of the data type and is a function of the methods, their arguments, their return values, and the *abstract* state of their target. We call the full set of commutativity rules for a data type its *commutativity specification.* An example specification for a *Set* is shown in Table \ref{tab:spec}. It is important to note that interface choice affects commutativity: if `add(x)` returned a boolean that expressed if the item was added or not, then `add(x)` would only commute with `add(y)` if $x \ne y$. In this case, the difference doesn't drastically impact commutativity, but it would affect the cost of checking commutativity dynamically.

For a given data type interface, there is one specification that expresses the maximum concurrency, like the one in Table \ref{tab:spec}, but there may be many that express varying degrees of commutativity. The space of possible specifications forms a *lattice* where $\top$ specifies the maximum possible commutativity, and $\bot$ represents the case where no operations are allowed to commute. As Kulkarni \cite{Kulkarni:PLDI11} explored, there is a complicated tradeoff between exposing more concurrency and efficiently tracking commutativity. Data structure designers and users navigate these choices to optimize their programs.

## Transactional Boosting
If two operations on the same record in two different transactions commute, then the transactions can safely execute concurrently, even though they both update the record. This technique of raising the level of abstraction in transactions to operations is known as *transactional boosting* \cite{Herlihy:PPoPP08}. This straightforward use of commutativity was shown to significantly improve abort rates in state-of-the-art software transactional memory. In Section \ref{sec:evaluation}, we show how it can be applied to distributed transactions.

## Other opportunities

Sometimes, due to heavily skewed workloads such as those arising from social networks, there may be so many requests to a single record that just executing all the operations, even without aborting, is a bottleneck which prevents scaling. The Ellen Degeneres selfie retweet is a prime example of this. In addition to transactional boosting, which we evaluate in this work, there are many other uses for commutativity. We mention just a few here which we have yet to evaluate.

### Record splitting for parallelism

Doppel \cite{Narula:OSDI14} observed that several contentious operations, such as incrementing counters or keeping track of the maximum bid, actually commute because the application doesn't need the output of each update operation. This allows them to *split* hot records onto multiple cores and execute commutative operations in parallel, *reconciling* them back to a single value before executing reads or non-commuting operations. This observation can be generalized via ADTs: operations can operate on copies of a record in parallel provided they commute with each other and provided they can be combined at the end of a phase before beginning to execute a new phase with a different set of operations that didn't commute with the first set.

### Combining

Another way to reduce contention on a shared data structure is to synchronize hierarchically: first with a couple immediate neighbors, then possibly with more clusters of neighbors, and finally with the data structure itself. This is known as combining \cite{flatCombining,yew:combining-trees,funnels} in the multi-processing literature, and could be applied to contended records in our model just as well as to shared data structures where it originated.
