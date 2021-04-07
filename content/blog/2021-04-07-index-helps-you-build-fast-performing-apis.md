---
layout: blog
title: Index  helps you build fast performing APIs
date: 2021-04-07T09:44:05.780Z
---
I was opportune to present this topic at engineering a few weeks ago, so I decided to share it on my blog as well. Database indexing is a huge subject, but here I have streamlined it to fit how we can have fast performing APIs with the right indexes.

**What is a Database Index?**

An index is a special lookup table that aids faster table searches, it also aids faster updates and deletes by a database engine.

An index in a database is similar to an index at the back of a book. for example, if we want to locate a topic called “Operating Systems”, let’s say this topic is on page 78 out of 100 pages in the book, without an index, chances are that we have to flip through each page of the book up till the 78th page before we can find the topic "Operating Systems". Even if we do not flip through each page, we will keep turning pages until we locate the topic, you agree this is stressful and wastes time right? but with an index, we will not flip through 77 pages to find the topic, all we need to do is go to the index of the book, locate the topic and its page number, then go back to the book and specifically get to that page number. Now you agree that this is a faster searching approach right? It is exactly the same thing that happens when we place an index on a database column. 

As an example, if we have a database table called Products that has 4 columns: Id, Name, Price, Quantity, let’s say this table has 40,000 records and we want to locate all the records that have Price as 15,000.

**Select *  from Products where price = 15000**

When we run the above query without an index, the database engine will return the records for us, right? But what the database engine has done under the hood is scan through 40,000 records and filter out the records that have their Price as 15,000, before returning them to us, This is called a table scan and table scan is bad performance. Imagine that the table has millions of records and the database has to do this table scan, it will take quite some time, and if we are loading this query from our application UI, it will pose a bad user experience because the page that calls the API would delay before it loads the data. With an index on the Price column, the database engine creates another special lookup table where it saves Prices with a row address of each price, such that when we run the above query, the database engine goes to that lookup table and retrieves the row address of that price, then uses the row address to fetch the records from the actual table, If there are 3 records with Price of 15,000, the lookup table will have those 3 records with their unique row addresses, it will start from the first occurrence, and then scan downwards to fetch all row addresses that match the criteria. Let’s illustrate this example.

*Products Table:*

![products table](/images/uploads/products-table.png "products table")

*Index:*

![](/images/uploads/index-table.png)

*Resulting records:*

![](/images/uploads/result-table.png)

### **When To Use Index?**

While creating an index, bear in mind that updating a table with indexes takes more time than updating a table without because the indexes also need an update. It's important to only create indexes when it is needed and absolutely necessary. The next question on your mind would be when is it absolutely necessary? The following are some valid reasons to create an index on a column:

1. When the column is queried frequently, by this it means when the column often appears in where clauses, or when the query that this column appears in is run very frequently, then the column should be indexed. 
2. Secondly, if a referential integrity constraint exists on the column, then the column needs an index on it, which means columns that would partake in joins in your query require an index.
3. If a UNIQUE key integrity constraint exists on the column, then the column needs a unique index on it. A unique index is a way of applying a unique key constraint on a column, the database engine would create the special lookup table (index) as well as enforce uniqueness of values in that column.

### **When To Not Use Index?**

Indexes are not needed in all scenarios and too many indexes are also not a good practice as this can cause issues, Some scenarios where we don’t need to add an index:

a. When the table is a small table for example a table holding records that don’t increase at all or don’t increase significantly, e.g a table that holds days in the week, months in the year, public holidays in a year, cities in a state. (some of these can be enums instead of tables anyway). These are small tables, so even if any column in these type of tables appears frequently in where clauses, there is still no need for an index, this is because indexes take up storage, and not indexing these columns will not hurt performance, since the table scan would be done on a minimal number of records.

b. Another scenario to not index is when there are low Reads and high Writes on the table, this means when the table is not often searched or queried but it is often updated, it will not hurt performance if no index is placed on the columns, regardless of the table size.

c. When we are often pulling a great percentage of the database records and not a small percentage. You don’t want to place an index on a column if, in your search criteria, it’s going to pull almost all the records in the table. This point may be debatable but check out the query below;

**Select * from Employees where id < 10000**

Assuming *id* increments by 1.

If the number of records is maybe 11,000 or even 12,000 and if this is all the query you need to run on this table, then there is no need to index because the database engine would have to read the index heavily to fetch all the records which is poor performance for the database. It will not hurt to do a table scan on 12,000 records if the expected result is 10,000 records.

**What Columns Should I Index?**

a. Index should be placed on columns that often appear in *where* clauses and *order by* clauses.

b. A column that has a unique constraint should have a unique index.

c. A column that is a foreign key needs an index.

**Single or Composite indexes:**

A single index is an index on one column and a composite index is an index on multiple columns.

**When to use both?**

We use a composite index when multiple columns are frequently being searched against and a single index when only one column is frequently being searched against. When creating a composite index, we need to specify the order of the index in each column, this order is used by the database engine to determine which column comes first, second, third.. in the special lookup table it is going to create, and helps us in determining how the *where* clause of the query should be structured. It is compulsory to order our columns in the *where* clause according to the same way the database engine orders the columns in the *lookup table*.

Understanding how to order indexes is very important in other not to waste storage space and memory in creating unnecessary or unused indexes, one may create an index on a column, that may appear in Where clauses, still the query may not benefit from the index. An example:

If you add an index on City and Country columns, if the index order of Country is 0 and City is 1, then the order of the columns in the where clause of a select, update or delete query on that table should be:

*Select Field_Name1, Field_Name2… from TableName where Country = “value” and City= “value”*

The following query will not benefit from that index, and in fact, that will be a waste of index, storage, and memory if the query was structured in this manner.

*Select Field_Name1, Field_Name2… from TableName where City= “value” and Country = “value”*

In the same manner, an index in the order Country (0), State(1), City(2) will benefit the following query

*Select Field_Name1, Field_Name2… from TableName where Country = “value” and State = “value” and City = “value”*

Or

*Select Field_Name1, Field_Name2… from TableName where Country = “value” and State = “value”*

Or

*Select Field_Name1, Field_Name2… from TableName where Country = “value”*

But it will not benefit the following query.

*Select Field_Name1, Field_Name2… from TableName where State = “value” and Country = “value”*

The reason is that column ordering matters when creating composite indexes, the column that has the least order number must always come first in the *where* clause, it should follow that order to the column with the highest order number, which means that City with column order 2 should come last in the *where*.

Also, with the above, keep in mind that there are different types of indexes applied on columns depending on what you want to achieve such as Clustered and Non-Clustered index, Filtered index, Unique index (I mentioned it in this article), etc.