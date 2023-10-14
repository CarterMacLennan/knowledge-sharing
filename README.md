# Open-Sourcing my Learning

- [Query Processing](#query-processing)
- [Data Modeling](#data-modeling)
- [Database Internals](#database-internals)

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

## Database Internals

##### Reference:
- [Database Internals](https://www.oreilly.com/library/view/database-internals)
- [Designing Data Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/)
