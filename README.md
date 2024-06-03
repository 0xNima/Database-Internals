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
<br/>

#### Memory- Versus Disk-Based DBMS
* **In-memory** database management systems (sometimes called main memory DBMS) store data primarily in memory and use disk for <ins>recovery</ins> and <ins>logging</ins>.
* **Disk-based** DBMS holds <ins>most</ins> of the data on disk and uses <ins>memory</ins> for <ins>caching disk contents</ins> or as a <ins>temporary storage</ins>.
* Accessing memory has been and remains several orders of magnitude faster than accessing disk.
* Benefits of using **mian memory** DBMS:
  - Performance
  - Comparatively low access costs
  - Access granularity
  - Simpler programming than on-disk DBMS
* Drawbacks of using **main memory** DBMS:
  - Main limiting factor is <ins>RAM volatility (lack of durability)</ins>
  - Cost
* There are ways to ensure durability but they require additional hardware resources and operational expertise.
* In practice, disks are **easier to maintain** and have significantly **lower prices**. However, the availability and popularity of **Non-Volatile Memory (NVM)** could change this situation.
* **NVM storage**: <ins>reduces</ins> or completely <ins>eliminates</ins> asymmetry between **read and write latencies**, <ins>improves</ins> **read and write performance**, and <ins>allows</ins> **byteaddressable access**.
<br/>

#### Durability in Memory-Based Stores
* In-memory database systems maintain **backups** on **disk** to provide durability and prevent loss of volatile data.
* An operation can be considered complete, **after** its results are written to a **sequential log file**.
* To avoid replaying complete log contents during startup or after a crash, in-memory stores maintain a **backup copy**:
  - Maintained as a **sorted disk-based structure**
  - Modifications to this structure are often **asynchronous**
  - Modifications applied in batches to reduce the number of I/O operations.
  - During recovery, database contents can be restored from the backup and logs.
* Log records are usually applied to backup in batches.
* After the batch of log records is processed, backup holds a database **snapshot** for a <ins>specific point in time</ins>, and <ins>log contents</ins> up to this point can be <ins>discarded</ins>. This process is called **checkpointing**.
* Disk-based databases with <ins>huge page cache</ins> still have limitations due to <ins>data layout</ins> and <ins>serialization</ins> in comparison with in-memory databases.
* Disk-based storage structures often have the form of wide and short **trees** while memory-based implementations can choose from a larger pool of data structures and perform optimizations that would otherwise be impossible or difficult to implement on disk.
<br/>

#### Column- Versus Row-Oriented DBMS
* **Column-Oriented**: Tables partitioned **Vertically** (storing values belonging to the same column together).
* **Row-Oriented**: Tables partitioned **Horizontally** (storing values belonging to the same row together).
* In the below image (a) shows the values partitioned column-wise, and (b) shows the values partitioned row-wise.

![Screenshot 2024-06-03 200559](https://github.com/0xNima/Database-Internals/assets/79907489/47511654-6b1d-4eb1-9c19-30d75d6a0df2)
<br/>

#### Row-Oriented Data Layout
* Row-oriented database management systems store data in records or rows.
* This approach works well for cases where several fields constitute the record (name, birth date, and phone number) uniquely identified by the key (in this example, a monotonically incremented number).
* Since row-oriented stores are **most useful** in scenarios when we have to access data by row, storing entire rows together improves **spatial locality**.
* **Spatial locality** is one of the Principles of Locality, stating that if a memory location is accessed, its nearby memory locations will be accessed in the near future.
* Data on a persistent medium such as a disk is typically accessed block-wise (in other words, a minimal unit of disk access is a block). So in row-oriented DBMS, if we would like to **access <ins>an</ins> entire user record**, because of **Spatial Locality**, a single block will contain data for all columns of that record, which is great for such a scenario. **On the other hand**, if we want to access to **individual fields** of **multiple user records** (like **phone number** of a user), it makes queries **more expensive**.
* Exampls: **PostgreSQL**, **MySQL**.
<br/>

#### Column-Oriented Data Layout
* Values for the same column are stored **contiguously** on disk. For example, if we store historical stock market prices, price quotes are stored together.
* Storing values for different columns in **separate files** or **file segments** allows **efficient queries by column**, since they **can be read in one pass** rather than consuming **entire rows** and discarding data for columns that weren’t queried.
* Column-oriented stores are a good fit for analytical workloads that compute aggregate, such as finding trends and computing average values.
<br/>

#### Distinctions and Optimizations
* Distinctions between row and column stores **are not** only in how the data is stored. Choosing the data layout is just one of the steps in a series of possible optimizations that columnar stores are targeting.
* Reading multiple values for the same column in one run significantly improves cache utilization and computational efficiency.
* Storing values that have the **same data type together** (e.g., numbers with other numbers, strings with other strings) offers a **better compression ratio**. Because we can choose the most efficient comparison algorithm for each data type.
* To decide **whether to use a column- or a row-oriented store**, we need to understand our **access patterns**.
  - If the read data is consumed in records (i.e., **most or all of the columns are requested**) and the workload consists mostly of **point queries** and **range scans**, the **row-oriented** approach is likely to give better results.
  - If scans **span many rows**, or **compute aggregate over a subset of columns**, it is worth considering a **column-oriented** approach.
* Examples: **MonetDB**, **C-Store**.
<br/>

#### Wide Column Stores
* <ins>Column-oriented</ins> databases **should not** be mixed up with <ins>wide column</ins> stores.
* **Wide Column Stores**: Data is represented as a **multidimensional map**, columns are grouped into **column families** (usually storing data of the <ins>same type</ins>), and <ins>inside each column family</ins>, data is stored **row-wise**.
* A canonical example from the *Bigtable* is a **Webtable**. The **Conceptual structure** of the Webtable is different from how it is implemented.
  - Conceptual structure:
    * ![Screenshot 2024-06-03 210528](https://github.com/0xNima/Database-Internals/assets/79907489/e3c78590-de49-4794-8a8f-8dbcef21a24a)
  - Physical Layout:
    * ![Screenshot 2024-06-03 210751](https://github.com/0xNima/Database-Internals/assets/79907489/34465537-1ffa-4890-a6ea-ca0e88aefc9b)
    * Column families are stored separately, but in each column family, the data belonging to the same key is stored together.
* Examples: **BigTable**, **HBase**.
