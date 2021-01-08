---
title: Create a Table
summary: Define tables for your database schema
toc: true
---

This page provides best-practice guidance on creating tables, with some simple examples based on Cockroach Labs' fictional vehicle-sharing company, [MovR](movr.html).

Recall that tables are the logical objects in a cluster that hold records of data in rows and columns. Before you start adding tables to your cluster, you need to [start a cluster](secure-a-cluster.html), and then [create a database and a user-defined schema](schema-design-database-schema.html).

{% include {{page.version.version}}/sql/dev-schema-changes.md %}

## Before you begin

Before reading this page, do the following:

- Start a [local CockroachDB cluster](secure-a-cluster.html), or create a [CockroachCloud cluster](cockroachcloud/create-your-cluster.html).
- Review [the database schema objects](schema-design-overview.html).

## Creating tables

To create a table, use a `CREATE TABLE` statement.

`CREATE TABLE` statements generally take the form:

~~~
CREATE TABLE <table_name> (
  <elements>
  );
~~~

Where the `table_name` is the name of the table, and `elements` is a comma-separated list of table elements, such as column definitions. We cover best practices for column definitions [below](#defining-columns).

As a general best practice when naming the table, use the fully-qualified name (e.g., `CREATE TABLE database.schema.table`).

For detailed reference documentation on the `CREATE TABLE` statement, including additional examples, see the [`CREATE TABLE` syntax page](create-table.html).

## Defining columns

After opening a `CREATE TABLE` statement and naming the table, you can start defining columns.

Column definitions generally take the following form:

~~~
<column_name> <DATA_TYPE> <column_qualification>
~~~

Where `column_name` is the column name, `DATA_TYPE` is the data type of the row values in the column, and `column_qualification` is some column qualification, such as a [constraint](#additional-constraints).

For example, if we want a table for users of the MovR platform, we could start with a few column definitions for the user's name and email:

~~~
CREATE TABLE cockroachlabs.movr.users (
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    email STRING UNIQUE
);
~~~

Any row value in the `first_name` column, for instance, must be of the data type `STRING`. The `NOT NULL` constraint also requires that the value not be a `NULL` value.

### Column definition best practices

Here are some best practices to follow when defining a column:

- Review the supported column [data types](data-types.html), and select the appropriate type for the data you plan to store in a column, following the best practices listed on the data type's reference page.
- Use column data types with a fixed size limit, or set a maximum size limit on column data types of variable size (e.g., `VARBIT(n)`). Values exceeding 1MB can lead to [write amplification](https://en.wikipedia.org/wiki/Write_amplification) and cause significant performance degradation.
- Review the best practices for [selecting primary key columns](#selecting-primary-key-columns), and decide if you need to define a dedicated primary key column.

## Selecting primary key columns

A primary key is a column, or set of columns, whose values uniquely identify rows of data. Every table requires a primary key.

Primary keys are defined in `CREATE TABLE` statements with the `PRIMARY KEY` constraint. The `PRIMARY KEY` constraint requires that all the constrained column(s) contain only unique and non-`NULL` values.

When a `CREATE TABLE` statement is executed, CockroachDB creates an index (called the `primary` index) on the column(s) constrained by `PRIMARY KEY` constraint. All values are automatically sorted by this index, which CockroachDB uses to find the rows in a table. For detailed information about how CockroachDB uses indexes, see [Indexes](indexes.html).

To constrain columns with the `PRIMARY KEY` constraint, add the `PRIMARY KEY` constraint to the column definition, or create the constraint in a separate constraint clause after the column definitions. For example, to create a `PRIMARY KEY` constraint on the `first_name` and `last_name` columns:

~~~
CREATE TABLE cockroachlabs.movr.users (
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    email STRING UNIQUE,
    CONSTRAINT "primary" PRIMARY KEY (first_name, last_name)
)
~~~

You could alternatively define a new column, and add the constraint directly to that column's definition. For example:

~~~
CREATE TABLE cockroachlabs.movr.users (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    email STRING UNIQUE
)
~~~

For detailed reference documentation on the `PRIMARY KEY` constraint, including additional examples, see the [`PRIMARY KEY` constraint page](primary-key.html).

### Primary key best practices

Here are some best practices to follow when selecting primary key columns:

- Understand how the data in the primary key column(s) is [distributed across a cluster](architecture/distribution-layer.html).

    Non-uniform data distributions can lead to hotspots on a single range, or cause [transaction contention](transactions.html#transaction-contention).

- Consider the [data types](data-types.html) of the primary key column(s).

    A primary key column's data type can determine where its row data is stored on a cluster. For example, some data types are sequential in nature (e.g., [`TIMESTAMP`](timestamp.html)). Defining primary keys on columns of sequential data can result in data being concentrated in a smaller number of ranges, which can negatively affect performance.

- Define a primary key for every table.

    If you create a table without defining a primary key, CockroachDB will automatically create a primary key over a hidden, [`INT`](int.html)-typed column named `rowid`. By default, sequential, unique identifiers are generated for each row in the `rowid` column with the [`unique_rowid()` function](functions-and-operators.html#built-in-functions). The sequential nature of the `rowid` values can lead to a poor distribution of the data across a cluster, which can negatively affect performance. Furthermore, because you cannot meaningfully use the `rowid` column to filter table data, the primary key index on `rowid` does not offer any performance optimization. This means you will always have improved performance by defining a primary key for a table. For more information, see our blog post [Index Selection in CockroachDB](https://www.cockroachlabs.com/blog/index-selection-cockroachdb-2/).

- Define primary key constraints over multiple columns (i.e., use [composite primary keys](https://en.wikipedia.org/wiki/Composite_key)).

    When defining composite primary keys, make sure the data in the first column of the primary key prefix is well-distributed across the nodes in the cluster. To improve the performance of [ordered queries](query-order.html), you can add monotonically increasing primary key columns after the first column of the primary key prefix. For an example, see [Use multi-column primary keys](performance-best-practices-overview.html#use-multi-column-primary-keys) on the [SQL Performance Best Practices](performance-best-practices-overview.html) page.

- For single-column primary keys, use [`UUID`](uuid.html)-typed columns with [randomly-generated default values](performance-best-practices-overview.html#use-uuid-to-generate-unique-ids).

    Randomly generating `UUID` values ensures that the primary key values will be unique and well-distributed across a cluster.

- Avoid defining primary keys over a single column of sequential data.

    Querying a table with a primary key on a single sequential column (e.g., an auto-incrementing [`INT`](int.html) column) can result in single-range hotspots that negatively affect performance. Instead, use a composite key with a non-sequential prefix, or use a `UUID`-typed column.

    If you are working with a table that *must* be indexed on sequential keys, use [hash-sharded indexes](hash-sharded-indexes,html). For details about the mechanics and performance improvements of hash-sharded indexes in CockroachDB, see our [Hash Sharded Indexes Unlock Linear Scaling for Sequential Workloads](https://www.cockroachlabs.com/blog/hash-sharded-indexes-unlock-linear-scaling-for-sequential-workloads/) blog post.

## Add secondary constraints

In addition to the `PRIMARY KEY` constraint, CockroachDB supports a number of other column-level constraints, including `CHECK`, `DEFAULT`, `FOREIGN KEY`, `NOT NULL`, and `UNIQUE`. Using constraints can simplify table queries, improve query performance, and ensure that data remains semantically valid.

To constrain a column, you can add the constraint to the column's definition, as shown in the `PRIMARY KEY` example above. To constrain more than one column, add the entire constraint's definition after the list of columns in the `CREATE TABLE` statement, as shown in the `PRIMARY KEY` example above.

For detailed reference documentation on a constraint, see [the constraint's syntax page](constraints.html).

### Populate with default values

To set default values on columns, use the `DEFAULT` constraint. Default values enable you to write queries without the need to specify values for every column.

When combined with [supported functions](functions-and-operators.html), default values can save resources in your application's persistence layer by offloading computation onto CockroachDB. For example, rather than using an additional library for generating unique ID's, you can set a default value to be an automatically-generated `UUID` value with the `gen_random_uuid()` function. Similarly, you could use a default value to populate a `TIMESTAMP` column with the current time of day, using the `now()` function.

For detailed reference documentation on the `DEFAULT` constraint, including additional examples, see [the `DEFAULT` syntax page](default-value.html).

### Reference other tables

To reference values in another table, use a `FOREIGN KEY` constraint. `FOREIGN KEY` constraints enforce [referential integrity](https://en.wikipedia.org/wiki/Referential_integrity), which means that a column can only refer to an existing column.

Note that foreign key dependencies can significantly impact query performance, as queries involving tables with foreign keys, or tables referenced by foreign keys, require CockroachDB to check two separate tables. We recommend using them sparingly.

For detailed reference documentation on the `FOREIGN KEY` constraint, including additional examples, see [the `FOREIGN KEY` syntax page](foreign-key.html).

### Prevent duplicates

To prevent duplicate values in a column, use the `UNIQUE` constraint. When you add a `UNIQUE` constraint to a column, CockroachDB creates a secondary index on that column.

Note that the `UNIQUE` constraint is implied by the `PRIMARY KEY` constraint.

For detailed reference documentation on the `UNIQUE` constraint, including additional examples, see [the `UNIQUE` syntax page](unique.html).

### Prevent `NULL` values

To prevent `NULL` values in a column, use the `NOT NULL` constraint. If you specify a `NOT NULL` constraint, all queries against the table with that constraint must specify a value for that column, or have a default value specified with a `DEFAULT` constraint.

Note that the `NOT NULL` constraint is implied by the `PRIMARY KEY` constraint.

For detailed reference documentation on the `NOT NULL` constraint, including additional examples, see [the `NOT NULL` syntax page](not-null.html).

## Add `CREATE` statements to your initialization file

Let's add some example tables to our `movr` schema.

We can use the same `dbinit.sql` file that you created in [Create a Database and Schema](schema-design-database-schema.html). We'll start with three tables: `users`, `vehicles`, and `rides`.

Open `dbinit.sql`, and, under the `CREATE SCHEMA` statement, add the `CREATE` statement for the `users` table:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.users (
    id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    email STRING UNIQUE
)
~~~

Now add a definition for the `vehicles` table.

This table needs to include information about vehicle's owner, the type of vehicle, when it was created, and what its status is.

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.vehicles (
      id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
      owner_id UUID NOT NULL REFERENCES cockroachlabs.movr.users(id),
      type STRING,
      creation_time TIMESTAMP(TZ),
      status ENUM,
      last_location STRING
  )
~~~

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.rides (
      id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
      rider_id UUID NOT NULL REFERENCES cockroachlabs.movr.users(id),
      vehicle_id UUID NOT NULL REFERENCES cockroachlabs.movr.vehicles(id),
      start_address STRING,
      end_address STRING,
      start_time TIMESTAMP(TZ),
      end_time TIMESTAMP(TZ),
      revenue DECIMAL(10,2)
  )
~~~

Before executing the statements in the file, we s

## See also


