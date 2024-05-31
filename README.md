# Database-Internals
A Diary of Reading Alex Petrov's Database Internals book 
<br/>
<br/>
## Day 1: 31-05-2024
### Preface
* horizontal scaling (scaling out) is better than vertical scaling (scaling up). Because it is easier to spin up a new instance and add it to the cluster than scaling up by moving the database to a larger, more powerful machine. Migrations can be long and potentially incur downtime.
* The most significant distinctions between database systems are concentrated around: how they store and distribute the data. So this book is arranged into parts that discuss the subsystems and components responsible for storage (Part I) and distribution (Part II).
<br/>

### Part I
### Storage Engines
* Databases are modular systems and consist of multiple parts: a transport layer accepting requests, a query processor determining the most efficient way to run queries, an execution engine carrying out the operations, and a storage engine.
* The **storage engine** is a software component of a database management system responsible for **storing**, **retrieving**, and **managing data** in memory and on disk, designed to capture a persistent, long-term memory of each node
* For flexibility, both keys and values can be arbitrary sequences of bytes with no prescribed form. Their sorting and representation semantics are defined in higher-level subsystems.
* Many storage engines were developed independently from DBMS and we can embed them in our projects. For example, MySQL has many storage engines like InnoDB, MyISAM, and RocksDB.

#### Comparing Databases
* Trying to compare databases based on their components (e.g., storage engine, replication, and distribution, etc.), their rank (based on arbitrary popularity value assigned by consultancy agencies or database comparison websites), or implementation language (C++, Java, or Go, etc.) can lead to **invalid and premature conclusions**. These methods can be used only for a high-level comparison and can be
as coarse as choosing between HBase and SQLite, so **even a superficial understanding of how each database works and what’s inside it** can help you land a more weighted conclusion.
* If you’re searching for a database that would be a good fit for the workloads you have, the **best thing you can do** is to **simulate these workloads against different
database systems**, **measure the performance metrics that are important for you**, and compare results.
* Some performance and scalability issues, start showing only after some time or as the capacity grows. To detect potential problems, it is **best to have long-running tests in an environment that simulates the real-world production setup** as closely as possible.
* Database choice is always a combination of these factors, and **performance** often turns out **not** to be **the most important** aspect: it’s usually **much better** to use a **database that slowly saves the data** than one that **quickly loses it**.
* To compare databases, it’s helpful to understand the use case in great detail and define the current and anticipated variables, such as:
  - Schema and record sizes
  - Number of clients
  - Types of queries and access patterns
  - Rates of the read and write queries
  - Expected changes in any of these variables

  Knowing these variables can help to answer the following questions:
  - Does the database support the required queries?
  - Is this database able to handle the amount of data we’re planning to store?
  - How many read and write operations can a single node handle?
  - How many nodes should the system have?
  - How do we expand the cluster given the expected growth rate?
  - What is the maintenance process?

* If there’s no standard stress tool to generate realistic randomized workloads in the database ecosystem, it might be a red flag
* **throughput**: the number of transactions the database system can process per minute.
* **TPC-C Benchmark**
  - The Transaction Processing Performance Council (TPC) has a set of benchmarks that database vendors use for comparing and advertising the performance of their products.
  - TPC-C is an online transaction processing (OLTP) benchmark, a mixture of read-only and update transactions that simulate common application workloads.

#### Understanding Trade-Offs
* When looking at different storage engines, we discuss their benefits and shortcomings. If there was an optimal storage engine for every conceivable use case, everyone would just use it. But since it does not exist, we need to choose wisely, based on the workloads and use cases we’re trying to facilitate.
* Similarly, design decisions made by storage engine developers make them better suited for different things: some are optimized for low read or write latency, some try to maximize density (the amount of stored data per node), and some concentrate on operational simplicity.
<br/>

### Chapter 1
### Introduction and Overview
* This chapter aims to disambiguate terminologies and find their precise definitions.
* Types of databases:
  - Based on **storage medium**: **Memory-** vs **Disk-Based**
  - Based on **layout**: **Column-** vs **Row-Oriented**
  - Other sourced group DBMS into three categories:
      + Online transaction processing (OLTP) databases:
        * These handle a large number of user-facing requests and transactions.
        * Queries are often predefined and short-lived
      + Online analytical processing (OLAP) databases
        * These handle complex aggregations.
        * OLAP databases are often used for analytics and data warehousing and are capable of handling complex, long-running ad-hoc queries.
      + Hybrid transactional and analytical processing (HTAP)
        * These databases combine properties of both OLTP and OLAP stores.
  - Other classifications that are not discussed in this book:
      + key-value stores
      + relational databases
      + document-oriented stores
      + graph databases
<br/>

#### DBMS Architecture
* Database management systems use a client/server model, where database system instances (nodes) take the role of servers, and application instances take the role of clients.

![Screenshot 2024-05-31 154732](https://github.com/0xNima/Database-Internals/assets/79907489/674f4726-6f7e-4981-8f82-9d2333be9da2)

* Client requests arrive through the *transport* subsystem. Requests come in the form of queries, most often expressed in some query language. The *transport* subsystem is also responsible for communication with other nodes in the database cluster.
* The *transport* subsystem hands the query over to a *query processor*, which **parses**, **interprets**, and **validates** it. Later, **access control checks** are performed, as they can be done fully only **after** the query is interpreted.
* The parsed query is passed to the *query optimizer*, which **first** <ins>eliminates</ins> **impossible** and **redundant** parts of the query, and **then** attempts to <ins>find the most efficient way</ins> to execute it <ins>based on internal statistics</ins> (index cardinality, approximate intersection size, etc.) and <ins>data placement</ins> (which nodes in the cluster hold the data and the costs associated with its transfer).
* The query is usually presented in the form of an *execution plan* (or *query plan*): a sequence of operations that have to be carried out for its results to be considered complete.
* Since the same query can be satisfied using **different execution plans** that can vary in efficiency, the **optimizer** picks the **best** available plan.
* The *execution plan* is handled by the *execution engine*, which collects the results of the execution of local and remote operations. *Remote execution* can involve writing and reading data to and from other nodes in the cluster, and replication.
* Local queries (coming directly from clients or from other nodes) are executed by the *storage engine*.
* The **storage engine** has several components with dedicated responsibilities:
  - **Transaction manager**: <ins>schedules transactions</ins> and <ins>ensures</ins> they cannot leave the database in a <ins>logically inconsistent state</ins>.
  - **Lock manager**: <ins>locks</ins> on the database objects for the running transactions, <ins>ensuring</ins> that <ins>concurrent</ins> operations <ins>do not violate physical data integrity</ins>.
  - **Access methods (storage structures)**: <ins>access</ins> and organizing data on disk. Access methods include **heap files** and storage structures such as **B-Trees**
  - **Buffer manager**: caches data pages in memory
  - **Recovery manager**: maintains the operation log and restores the system state in case of a failure
  - **Transaction** and **Lock Managers** are responsible for **concurrency control**. They guarantee <ins>logical and physical data integrity</ins> while ensuring that concurrent operations are executed as efficiently as possible.
