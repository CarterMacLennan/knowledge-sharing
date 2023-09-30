# Open-Sourcing my Learning

- [Data Modeling](#data-modeling)
- [Query Processing](#query-processing)
- [Distributed Systems](#distributed-systems)
- [Database Internals](#database-internals)

## Data Modeling
#### Introduction

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

#### Recommendations
- [Data Warehouse Toolkit](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/data-warehouse-dw-toolkit/)

## Distributed Systems
#### Recommendations
- [Designing Data Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/)
- [System Design Interview Books from ByteByteGo](https://blog.bytebytego.com/p/system-design-interview-books-volume)

## Query Processing
#### Introduction

#### Recommendations
- [Designing Data Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/)

## Database Internals
#### Introduction

#### Recommendations
- [Database Internals](https://www.oreilly.com/library/view/database-internals)
- [Designing Data Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/)
