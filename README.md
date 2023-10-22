# Open-Sourcing my Learning

- [Database Internals](#database-internals)
- [Distributed Systems](#distributed-sytems)
- [Data Modeling](#data-modeling)
- [Query Processing](#query-processing)

## Database Internals

### Notes from "Database Internals" by Alex Petrov:

#### Storage Engines (or database engines)

Databases are modular systems made up of multiple parts:
1. Transport Layer accepting requests.
2. Query Processor for optimizing the way we run queries.
3. Execution Engine for carrying out the query.
4. Storage engine for storing, retrieving, and managing data in memory and on disk.

One good way of thinking about Storage Engines is to treat DBMS like applications that are built on top of storage engines. These applications offer a schema, query language, indexing, transactions, etc… On the other hand, the storage engines themselves are a software component that offer a simple data manipulation API allowing CRUD actions. 

It’s not uncommon for storage engines to be developed independently from their DMBS they become embedded into (i.e., BerkeleyDB, LevelDB, RocksDB, LMDB,etc….). This approach has enabled devs to focus on enhancing/ extending other subsystems by using pluggable database engines which can be swapped out for different use cases (i.e., MySQL has several storage engines: RocksDB, InnoDB, etc…). 

**CHAPTER 1 - Introduction & Overview**

Three Major Categories:
- OLTP: Handle large number of user-facing requests and transactions. Queries often predefined and short-lived.
- OLAP: Used for analytics (complex aggregations) and data warehousing, they are capable of handling complex, long-running ad hoc queries. 
- HTAP: Combine properties of both OLAP and OLTP.

There are many other terms and classifications: 
- Key-Value Stores
- Relational Databases
- Document-Oriented Stores
- Graph Databases

DBMS can also serve different purposes: 
- Temporary hot data Vs.  long-lived cold storage
- Complex analytical queries Vs.  accessing values by a given key
- Optimized to store time-series data Vs. storing large blobs efficiently

**DBMS Architecture:**

<img width="608" alt="Screenshot 2023-10-15 at 11 08 12 AM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/6e54372e-f3ea-4e41-bceb-060dc2d80d41">

**Transport Subsystem:**

1. Receives client requests in the form of queries (expressed in some query language). 
    - Forwards queries to the _query processor_.
2. Responsible for communication between nodes in the database cluster.

**Query Processor:**

1. Query Parser: Parses, interprets, and validates the query. After interpreting, access controls checks are performed.
2. Query Optimizer: Eliminates redundant parts of the query, finds optimal way to execute based on internal statistics and data placement.
3. Outputs the best available _execution plan/ query plan_ (list of operations to execute the query).

**Execution Engine:**

Collects the results of the execution of local & remote operations.
- Remote Execution: Reading/ writing data to and from other nodes in the cluster.

**Storage Engine:**

- Transaction Manager: Schedules transactions & ensures they leave in a consistent state.
- Lock Manager: Ensures concurrent operations don't violate data integrity by locking onto database objects for the currently running transactions.
- Access Methods: The storage structures that manage accessing and organizing data on disk.
  - Heap files, B-Trees, LSM Trees, etc...
- Buffer Manager: Caches data pages in memory.
- Recovery Manager: Maintains the operation log and restoring the system state in case of failure.

**Memory Vs. Disk-Based DBMS**

Disk-based databases: 
- Hold most of the data on **disk**.
- Use memory for caching disk contents or as temporary storage.

In-memory databases (also known as "main memory databases"):
- Store data primarily in **memory**
- Use **disk** for recovery and logging.

Programming for Main Memory Vs. Disk-Based:
In-memory databases use different data structures, organization, and optimization techniques from their disk-based counterparts. The benefit here is that programming for main memory is much simpler than for disk-based databases. This is because the OS abstracts memory management away from us to allow us to think in terms of allocating and freeing memory chunks. Main memory databases can choose from a larger pool of data structures that would be impossible or very complex on disk. They're also easily able to handle variable-size data by simply referencing the value with a pointer. On the other hand, when working on disk we have to think about: managing data references, serialization formats, freed memory, and fragmentation. They also use specialized storage structures, optimized for disk access (i.e., wide and short trees).

Limitations of Main Memory Databases:

1. RAM Volatility: Since RAM does not persist data, software crashes, hardware failures, and power outages result in data loss. To mitigate these risks you can ensure uninterrupted power supplies or battery-packed RAM but these require additional up-front and maintenance cost.
2. Performance & Price Trade-Off: Naturally, accessing memory is several orders of magnitude faster than disk. However, RAM is still much more expensive than SSDs and HDDs. As memory prices go down they become a compelling alternative. On top of this, there is a rising popularity of Non-Volatile Memory which are also capable of improving read and write performance.

**Durability in Memory-Based Stores:**

Some main memory databases provide no durability guarentees by storing data exclusively in memory. However, most will maintain backups on disk to provide some durability guarentees. 

Before operations are marked complete, their results are written to a sequential log file. To avoid replaying the complete log contents after a crash in-memory stores maintain a "backup copy". This backup copy is maintained as a sorted disk-based structure, and modifications to this
structure are typically asynchronous and applied in batches to reduce the number of I/O operations. When these log records are applied in batches, the backup now holds a complete snapshot up to that specific point in time. Because of this, the log contents up to this point can be discarded. This is a technique known as "checkpointing".

**Column Vs. Row-Oriented DBMS**

Column-oriented DBMS partition data vertically (i.e., by column) instead of storing it in rows. As a result, values in the same column will be stored contiguously on disk (rather than by row). 

By storing different columns in separate file/ file segments we only have to read in the data we're querying. Otherwise we would be consuming entire rows and discarding the columns we don't need. Column-oriented databases are great for analytical workloads performing complex aggregations. 

One added benefit of Column-Oriented databases is they store values with the same data type together. This offers a better compression ratio since we can use different compression algorithms depending on the data type, allowing us to choose the most effective method for every case.

EXAMPLE:

```
| ID | Symbol | Date        | Price     |
| 1 | DOW    | 08 Aug 2018 | 24,314.65 |
| 2 | DOW    | 09 Aug 2018 | 24,136.16 |
| 3 | S&P    | 08 Aug 2018 | 2,414.45  |
| 4 | S&P    | 09 Aug 2018 | 2,232.32  |
```

BECOMES:

```
Symbol: 1:DOW; 2:DOW; 3:S&P; 4:S&P
Date: 1:08 Aug 2018; 2:09 Aug 2018; 3:08 Aug 2018; 4:09 Aug 2018
Price: 1:24,314.65; 2:24,136.16; 3:2,414.45; 4:2,232.32
```

Handling Tuples:
When we perform joins, filtering, and multirow aggregates we often require data tuples. To facilitate this, we need to preserve metadata on the column level to find the other data points in other columns it's associated with. 

1. Explicit: Each value holds a key. However, this greatly increases the amount of data stored.
2. Implicit: Use the position of the value (i.e., it's offset) to map to the associated value.

#### Data Files and Index Files

A database system usually separates data files and index files: 
- Data files store data records
- Index files store record metadata and use it to locate records in data files. 

Database systems store data records in tables usually represented as a separate file. In these tables records can be looked up using a search key alongside an index to avoid scanning the entire table. Files are partitioned into pages that are one or several disk blocks in size.

Inserts/ updates to existing records are represented by key/value pairs. Modern databases rarely explicty delete a page. Instead the simply use deletion markers known as "tombstones" that contain delete metadata (i.e., key & timestamp). This space is later reclaimed during garbage collection by reading the pages, writing live records to their new location and deleting the legacy records.

**Data Files**

Data files (also called primary files) can be implemented three different ways:
1. Heap-organized Tables (Heap Files)
2. Hash-organized Tables (Hashed Files)
3. Index Organized Tables (IOT)

Heap-organized Tables (Heap Files):
To avoid added work, records in heap files don't need to be inserted in a particular order. So, they're usually just placed in a write order. To make heap files searchable they require additional index structures to point to the locations where data records are stored.

Hash-organized Tables (Hashed Files):
Here, records are stored in buckets, and we use the hash value of the key to find the bucket containing our record. The records inside each bucket can be stored in append order or sorted by key to improve lookup speed.

Index Organized Tables (IOT):
This approach actually stores the data records in the index itself. Because records are already stored in key order, range scans in IOTs can be implemented by sequentially scanning its contents. This also reduces the number of disk seeks by at least one, since after locating the searched key, we don't need to address a separate file to find our data record.

**Index Files**

An index is a **structure that organizes data records** on disk in a way that **facilitates** **efficient** **retrieval** operations.They map keys to locations in data files where the records identified by these keys (in the case of heap files) or primary keys (in the case of IOT) are stored.

Primary Index:
An index on a primary (data) file is called the primary index. However, we can usually assume this primary index is built over a primary key(s). All other indexes are called secondary indexes.

Secondary Index:
Secondary indexes can point directly to the data record or just store its primary key. A pointer to a data record can hold an offset to a heap file or an index-organized table. 

Multiple secondary indexes can actually point to the same record, which allows a single data record to be identified by different fields and located through different indexes. 

Clustered Index:
An index is "clustered" if the order of **data records follows the search key order**. Data records in this case are typically **stored in the same file** or a "clustered file", so the key order is preserved. **Index-organized tables** store information in index order and **are clustered by definition**. Primary indexes are most often clustered.

Nonclusted/ unclustered Index:
Nonclustered Indexes are when the data is stored in a **separate file**, and its **order does not follow the key order**. **Secondary indexes are nonclustered** by definition, since they’re used to facilitate access by keys other than the primary one. Clustered indexes can be both index-organized or have separate index and data files.

**Buffering, Immutability, and Ordering**

A storage engine is based on some data structure. However, these data structures **do not include**:
- caching
- recovery
- transactionality
- other things that storage engines add on top of them....

Storage structures have 3 common variables for distinctions and optimizations in storage structures: 
1. They use buffering (or avoid using it), 
2. Immutable (or mutable) files
3. Store values in order (or out of order).

BUFFERING:
Should the storage structure collect "X" amount of data in memory before putting it on disk?
- Note: Every on-disk structure has to use buffering to some degree, as the smallest unit of data transfer to/ from disk is a block (and it is desirable to write full blocks). 

What we’re discussing is **avoidable buffering**. 
- Adding in-memory buffers to B-Tree nodes to amortize I/O costs (“Lazy B-Trees”).
- Two-component LSM Trees use buffering in an entirely different way, and combine buffering with immutability.

MUTABILITY (or immutability):
- Mutable: Should the storage structure read parts of a file, update then, and then simply write the updated part back at the same location in the file? 
- Immutable: Or should we append modifications at the end of the file and avoid modifying the contents of the file?

Alternative implementations of immutability:
- Copy-on-write: The modified page holding the updated version of the record, is written to a new location in the file, instead of its original location.
- Often the distinction between LSM and B-Trees is drawn as immutable against in-place update storage (mutable), but there are structures (i.e., BwTrees) that are inspired by B-Trees but are immutable.

ORDERING:
Should the data records be stored in key order in the pages on disk? This not only helps locate the individual data records but helps us efficiently scan the range of records. This is because the keys that "sort closely" are stored in contiguous segments on disk. 

Storing data out of order (i.e., in insertion order) opens up for some write-time optimizations. 
- i.e., Bitcask and WiscKey store data records directly in append-only files.

##### Reference:
- [Database Internals](https://www.oreilly.com/library/view/database-internals)
- [Designing Data Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/)

## Distributed Systems

_in progress_

## Data Modeling

### Normalization

Normalized tables can be easier to understand, enhance, and extend, as well as protect data integrity by preventing insertion/ update/ deletion of anomalies. To assess the level of danger we are facing due to a lack of normalization we have a set of criteria that categorizes the level of normalization of your database.

#### 1NF:

- Every cell in a table should only contain a single value that cannot be divided (i.e., cannot contain a list of names).
- Unique column names.
- The order in which data is stored in the database does not matter. For example, the rows shouldn't be expected to be stored in alphabetical order.
- Every row must be unique.

#### 2NF:

- First, the table must already be in 1NF.
- Every non-key attribute must depend on the entire primary key (i.e., composite key).
  - We should create a seperate table when we encounter attributes that do not meet meet the above criteria.

<img width="1167" alt="Screenshot 2023-10-14 at 3 51 52 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/857df34c-baaa-45b6-b306-c78fad4ec094">
<img width="954" alt="Screenshot 2023-10-14 at 3 51 35 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/16ad4c03-1ffe-43a4-b45c-6e302f03e0b1">


#### 3NF:

_Every non-key attribute in a table should depend on the key, the whole key, and nothing but the key._

- First, the table must already be in 2NF.
- Here we must get rid of any "transitive" dependencies (when non-primary keys depend on other non-primary keys).
  - To fix this we simply seperate this transitive dependency into a seperate table. 

<img width="799" alt="Screenshot 2023-10-14 at 3 51 13 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/3da9ae86-f538-4f3b-b047-bc76d864cc4b">
<img width="1201" alt="Screenshot 2023-10-14 at 3 50 54 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/1acac802-3156-4fcc-b06e-e01be765bbe2">

#### 4NF:

- First, the table must already be in 3NF.
- Here we want to ensure there are only multivalued dependencies on the key.
  - To do this we seperate these dependencies out into separate tables

<img width="457" alt="Screenshot 2023-10-14 at 3 50 22 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/daefa7a7-35f1-4dd9-b4a1-033afd438279">
<img width="871" alt="Screenshot 2023-10-14 at 3 49 51 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/11e5a6f4-20f8-452b-b11d-e41e65e181fd">


#### 5NF:

- First, the table must already be in 4NF.
- The table must not be able to be decomposed into any number of smaller tables without loss of data.

<img width="509" alt="Screenshot 2023-10-14 at 3 48 46 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/0173e712-73a0-4f7c-9c74-23a47e7a560c">
<img width="895" alt="Screenshot 2023-10-14 at 3 49 23 PM" src="https://github.com/CarterMacLennan/knowledge-sharing/assets/60050873/745f06bb-dab4-4db3-9b48-375793c11ef0">


### Inmon Data Modeling

The Inmon approach takes advantage of normalization to avoid data redundancy as much as possible. Additionally, this greatly simplifies data loading and helps prevent any data update irregularities. However, this approach does result in more complex queries which leads to less optimal query performance.

Inmon proposes constructing seperate data marts for each enterprise division (i.e., marketing) while the data warehouse acts as the single data source to ensure integrity and consistency.

### Kimball's Dimensional Modeling
Kimball's approach denormalizes the source/ staging data into two main components:

**Facts:**

Facts represents a business measure. They allow us to store low-level quantitative metrics from a business process. Since the fact table is overwhelmingly the largest set of data, whenever possible we should strive for facts that are both numeric and additive as often the most useful thing to do with that many rows is add them up. Semi-additive facts such as account balances prevent you from being able to sum across the time dimension, while non-additive facts like unit prices can't be added at all. These types of metrics force us to use counts, averages, or printing them independently (which doesn't provide much valule at scale). To ensure measurements aren't double-counted, it's critical that all the rows in your fact table must be at the same "grain" (single level of detail). The most granular or atomic data is preferred as data that has not been aggregated tends to be the most expressive and best at answering the often unpredictable ad-hoc queries from business users.

**Dimensions:**

Dimensions are awesome as they bring context/details (who, what, where, when, why, how) to our business measurements. They are also the source of the constraints on our queries (`where cereal = "Lucky Charms"`), allow us to make groupings (`group by manufacturer`), and report labels (i.e., column names). It's often said that a Data Warehouse is only as good as the quality, depth, and diversity of it's dimension tables. 

**Star Schema:** 

With the kimball approach every business process is represented by a dimensional modal with a single fact table surrounded by dimension tables. The fact table's primary key is a composite key composed of two or more foreign keys that link to the surrounding dimension table's primary keys. This approach tends to construct a star-like schema giving it the aptly named nickname "star schema". This simple design not only makes it easier due to it's universal approach, it makes it simpler for business users to navigate and helps database optimizers who favor simple schemas with few joins. 

**Snowflake Schema Vs. Star Schema**

Dimension tables tend to represent hierarchical relationships. This could be something like having columns: `address`, `city`, `state`. To benefit business users and improve query performance Kimball suggests to store these hierarchical attributes redundantly rather than normalizing the data. This normalization is nicknamed "snowflaking" due to the snowflake-like schema which results from using seperate look up tables to reduce redundancy.

**Important Fact Table Techniques**

_Table Structure_

- Foreign  Keys for each associated dimension table.
- Optional degenerate dimension keys (dimensions that didn't require an external table).
- Optional date/time stamps.
- Numeric measurements based on physical activity.

_Additive, Semi-Additive, Non-Additive Facts_

- Additive: Can be summed across ANY dimension associated with the fact table.
- Semi-Additive: Can be summed across some dimensions - but not all (i.e., time).
- Non-Additive: Cannot be added across any dimension (i.e., ratios).
  - The best practice here is to store the additive components and then sum + calculate in the BI layer.

_Working with Nulls_

- Null-valued measurements are acceptable as aggregate functions know how to deal with nulls correctly.
- We can't have null Foreign Keys as that would cause a referential integrity violation.
  - The best practice here is that the dimension table should have a default row representing the NA (Not Applicable) condition.
 
_Conformed Facts_

If the same fact appears in seperate fact tables we should:

1. If the facts are identical if they are compared together -> it is a "conformed fact" and they should be identically named.
2. If the facts are NOT identical if they are compared together -> they should be named differently to avoid them being computed together.

_[FACT TABLE TYPE I] Transaction Fact Table:_

Each row represents a measurement event at a point in time. Atomic grain transaction fact tables are the most expressive and best handle business user's unpredictable ad-hoc queries. These tables can be either dense or sparse as a row is only appended if a measurement event actually takes place (i.e., someone buys a lemonade from a lemonade stand).

_[FACT TABLE TYPE II] Periodic Snapshot Fact Table:_

A row here corresponds to many measurement events that occurred over a standard period (i.e., weekly, bi-weekly, monthly, etc...). These tables are uniformly dense as a row will be added regardless if a measurement event ever took place. 

_[FACT TABLE TYPE III] Accumulating Snapshot Fact Table:_

A row here corresponds to many measurement events that occurred at predictable steps between the start and end of a process (i.e., workflow process). A date forign key is used for each critical milestone in the process. As steps are reached rows in the fact table are **updated** (these repetitive updates is unique to this fact table type). Additionally, milestone completion counters and numeric lag measurements are often included.

_[FACT TABLE TYPE IV] Factless Fact Tables_

Often there is no numerical result to record from a measurement event. Instead we simply insert a row to record the "fact" that several dimensional entites came together at a moment in time. 

_[FACT TABLE TYPE V] Aggregate Fact Tables_

Aggregate fact tables can be used to accelerate query performance by rolling up atomic fact table data. However, they should behave like database indexes by accelerating queries without being used directly by business users/ BI apps. 

_[FACT TABLE TYPE VI] Consolidated Fact Tables_

Consolidated fact tables combine facts from multiple business processes together to simplify and optimize comparing cross-process metrics that are routinely analyzed together. 

Note:
1. We are off-loading complexity from the BI layer to the ETL layer.
2. It's critical that the facts must be expressed at the same grain.

**Important Dimension Table Techniques**

_Table Structure_

Dimension tables have a single primary key that acts as the foreign key in the relevant fact table(s). Typicallty dimension tables are wide and short while embracing denormalization to favour simplicity and query performance. On top of this because dimension table attributes are the target for report labels we want to use verbose descriptions while avoiding operational codes and acronyms.

_Dimension Surrogate Keys_

In dimension tables we want to avoid using natural keys as our primary keys. This is because natural keys for a dimension are subject to business rules out of our control, can be incompatible, or simply poorly administrated. To claim control over our primary keys we should create dimension surrogate keys for each dimension table (except for the date dimension). These are anonymous integers that are assigned in sequence starting with the value 1.

_Durable Supernatural Keys_

Whenever we want a new key (i.e., for an employee) we need to create a new “durable” key that will not change even if they resign and are subsequently rehired. Although we may have multiple surrogate keys over their lifetime, this new durable supernatural key will remain the same. Ideally, they should be independent of the business process and are simple integers assigned in sequence starting with 1.

_Degenerate Dimensions_

These dimensions have no content outside of their primary key. As a result the degenerate dimension is placed inside of the fact table with no associated dimension table.

_Multiple Hierarchies in Dimensions_

Many dimensions have a natural hierarchy like location (i.e., city, state, country). We should resist normalizing our dimensions and instead keep them in the same dimension table for simplicity and improved query performance.

_Null Attributes in Dimensions_

We end up will null attributes when a dimension row is not fully populated. Unlike  Fact Tables we want to substitute these nulls with a “Unknown” or “Not Applicable” string. This is because different databases will handle grouping/ constraining on nulls differently.

_Calendar Date Dimensions_

Most fact tables have a calendar date dimension. This is because rather than computing it in SQL you can simply look it up in the given dimension table. Note that in this case the primary key of a date dimension can be something like YYYYMMDD, rather than a sequentially-assigned surrogate key.

_Role-Playing Dimensions_

In some cases you will want to refer to a single physical dimension multiple times in a fact table (i.e., multiple dates in the date dimension). In this case you will have unique attribute columns (nicknamed "roles") that will enable seperate foreign keys to refer to these views.

_Junk Dimensions_

Often it can be handy to have a "junk dimension" to combine several miscellaneous lowcardinality flags that are produced by a business process rather than making seperate dimension tables for each one.

_Outrigger Dimensions_

Outrigger dimensions are dimensions that contain a reference to another dimension table. We can do this, however, typically we would want to show correlation between dimensions using seperate foreign keys in the corresponding fact table. One example of this would be a employee dimension having the attribute "first day" that references the calendar dimension.

**Slowly Changing Dimensions**

_Type 0: Retain Original_

These dimension attribute values never change so we don't have to worry about anything here. 

_Type 1: Overwrite_

These dimension attributes will actually change. However, we don't care about storing any historical here so we can simply overwrite the old value with a new one. 
- Note: When using this approach it's important to ensure that the aggregate fact tables that will be affected by this change are rolled up again.

_Type 2: Add New Row_

For type 2 dimensions we actually care about retaining the historical data. To do this, when an attribute changes we add a new row to the table with a new primary key. To maintain a connection between the two rows we allow the rows to retain the same natural key.

To enable us to analyze Type 2 Dimensions we can use one of the two following approaches:
1. A flag column to show which record is up-to-date (i.e., active).
2. A timestamp column to show when the record was added. Additionally, you could have two timestamp columns: one to mark when the record was up-to-date and when it was made out-of-date.

_Type 3: Add New Attribute_

With Type 3 dimensions we track changes by simply adding a new column. This is useful if you require your primary keys to remain unique and have just one record for each natural key. However, this doesn't scale as well since you can only really track one change in a record rather than multiple changes over time (using Type 2 Dimensions).

_Type 4: Add Mini-Dimension_

With Type 4 dimensions we store records in two separate tables: a current record table as well as a historical record table. Records that are up-to-date (active) will be in one "current" table while the historical records will be stored in the "historical" table. This scales well and works great for attributes that change often.

##### Reference:
- [Decomplexify "Learn Database Normalization"](https://www.youtube.com/watch?v=GFQaEYEc8_8&ab_channel=Decomplexify)
- [Data Warehouse Toolkit](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/data-warehouse-dw-toolkit/)

## Query Processing
#### Execution Order
Database systems create query execution plans to better optimize our queries. If we want to write performant queries it's important to understand how our queries will be executed. 

1. First you can imagine that we pull whatever table is requested in our `FROM` clause. If requested, we `JOIN` with the specifed table using the shared join columns provided.
3. Afterwards the `WHERE` clause is used to filter out rows that don't meet the conditional statement(s).
4. Next, the `GROUP BY` and `HAVING` clauses are executed.
5. Despite coming first in our queries we finally transition to the `SELECT` clause to define what columns we want to return.
6. Finally we end with out `ORDER BY` and `LIMIT` clauses

### Query Optimizations
**General Optimizations**

- SELECT only the columns you need (i.e., avoid `*` when possible).
- Use the LIMIT when wanting to preview your results. Good when running adhoc queries with pay-per-query billing (i.e., AWS Athena).
- Use wildcards only at the end of a phrase.
- Avoid SELECT DISTINCT when possible as it groups the results (which is expensive). You could simply select enough fields to give you unique results.
- Run expensive queries during off-peak hours.
- Try to limit your results using filtering and pagination. 

**Indexes**
- `SELECT` Clause: It's important to choose covering indexes which cover all the columns used in our SELECT, JOIN, and WHERE clauses.
- `JOIN` Clause: To improve the performance on your join, it's often beneficial to use indexes on your join columns.
- `ORDER BY` Clauses: To reduce the amount of data stored in memory and improve sorting performance it's important to once again use appropriate indexes.
- `WHERE` Clause: Here it's important to write SARGABLE queries as well - see below.

**SARGABLE Queries**

It's important to write SARGABLE (Search ARGument ABLE) queries which can use your indexes to improve your query performance. For example if you're using a function on an indexed column inside of your where clause it can prevent the underlying database engine from using the index on that column. This is because that function would have needed to have been applied on every row in that index. If this is necessary, many database systems support creating function-based indexes. This may be a valuable workaround depending on your query patterns.

##### Reference:
- [ByteByteGo Secret To Optimizing SQL Queries - Understand The SQL Execution Order](https://www.youtube.com/watch?v=BHwzDmr6d7s&ab_channel=ByteByteGo)
- [Cody Baldwin - SQL Query Optimization - Tips for More Efficient Queries](https://www.youtube.com/watch?v=GA8SaXDLdsY&ab_channel=CodyBaldwin)

