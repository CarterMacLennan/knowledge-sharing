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

#### Kimball's Dimensional Modeling
Kimball's approach denormalizes the source/ staging data into two main components:

**Facts:**

Facts represents a business measure. They allow us to store low-level quantitative metrics from a business process. Since the fact table is overwhelmingly the largest set of data, whenever possible we should strive for facts that are both numeric and additive as often the most useful thing to do with that many rows is add them up. Semi-additive facts such as account balances prevent you from being able to sum across the time dimension, while non-additive facts like unit prices can't be added at all. These types of metrics force us to use counts, averages, or printing them independently (which doesn't provide much valule at scale). To ensure measurements aren't double-counted, it's critical that all the rows in your fact table must be at the same "grain" (single level of detail). The most granular or atomic data is preferred as data that has not been aggregated tends to be the most expressive and best at answering the often unpredictable ad-hoc queries from business users.

**Dimensions:**

Dimensions are awesome as they bring context/details (who, what, where, when, why, how) to our business measurements. They are also the source of the constraints on our queries (`where cereal = "Lucky Charms"`), allow us to make groupings (`group by manufacturer`), and report labels (i.e., column names). It's often said that a Data Warehouse is only as good as the quality, depth, and diversity of it's dimension tables. 

**Star Schema:** 

With the kimball approach every business process is represented by a dimensional modal with a single fact table surrounded by dimension tables. The fact table's primary key is a composite key composed of two or more foreign keys that link to the surrounding dimension table's primary keys. This approach tends to construct a star-like schema giving it the aptly named nickname "star schema". This simple design not only makes it easier due to it's universal approach, it makes it simpler for business users to navigate and helps database optimizers who favor simple schemas with few joins. 

**Snowflake Schema Vs. Star Schema**

Dimension tables tend to represent hierarchical relationships. This could be something like having columns: `address`, `city`, `state`. To benefit business users and improve query performance Kimball suggests to store these hierarchical attributes redundantly rather than normalizing the data. This normalization is nicknamed "snowflaking" due to the snowflake-like schema which results from using seperate look up tables to reduce redundancy.

##### Reference:
- [Data Warehouse Toolkit](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/data-warehouse-dw-toolkit/)

## Database Internals

##### Reference:
- [Database Internals](https://www.oreilly.com/library/view/database-internals)
- [Designing Data Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/)
