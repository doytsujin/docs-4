---
title: Create a Table
summary: Define tables for your database schema
toc: true
---

This page provides best-practice guidance on creating tables, with some simple examples based on Cockroach Labs' fictional vehicle-sharing company, [MovR](movr.html).

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `CREATE TABLE` statement, including additional examples, see the [`CREATE TABLE` syntax page](create-table.html).
{{site.data.alerts.end}}

<div class="filters filters-big clearfix">
  <button class="filter-button" data-scope="local">Local</button>
  <button class="filter-button" data-scope="cockroachcloud">CockroachCloud</button>
</div>

## Before you begin

Before reading this page, do the following:

- [Install CockroachDB](install-cockroachdb.html).
<div class="filter-content" markdown="1" data-scope="local">
- Start a [local CockroachDB cluster](secure-a-cluster.html).
</div>
<div class="filter-content" markdown="1" data-scope="cockroachcloud">
- Create a [CockroachCloud cluster](cockroachcloud/create-your-cluster.html).
</div>
- Review [the database schema objects](schema-design-overview.html).
- Create [a database](schema-design-database.html).
- Create [a user-defined schema](schema-design-schema.html).

## Create a table

Tables are the [logical objects in a cluster](schema-design-overview.html#database-schema-objects) that store data sent from your application's persistence layer. Tables organize records of data in rows and columns.

To create a table, use a [`CREATE TABLE` statement](create-table.html), following the best practices that we have listed in the following sections:

- [Name a table](#name-a-table)
- [Define columns](#define-columns)
- [Select primary key columns](#select-primary-key-columns)
- [Add additional constraints](#add-additional-constraints)

### Name a table

`CREATE TABLE` statements generally take the form:

~~~
CREATE TABLE [schema_name].[table_name] (
  [elements]
  );
~~~

Where `schema_name` is the name of the user-defined schema, `table_name` is the name of the table, and `elements` is a comma-separated list of table elements, such as column definitions.

#### Table naming best practices

Here are some best practices to follow when naming tables:

- Use a fully-qualified name (i.e., `CREATE TABLE schema_name.table_name`). If you do not specify the user-defined schema in the table name, CockroachDB will create the table in the preloaded `public` schema.
- Use a table name that reflects the contents of the table. For example, for a table containing information about orders, you could use the name `orders` (as opposed to naming the table something like `table1`).

#### Table naming example

Suppose you want to create a table to store information about users of the [MovR](movr.html) platform.

In a text editor, open the `dbinit.sql` file that you used to create the `movr` user-defined schema in [Create a User-defined Schema](schema-design-schema.html).

Under the `CREATE SCHEMA` statement, add an empty `CREATE TABLE` statement for the `users` table. The file should now look something like this:

{% include copy-clipboard.html %}
~~~
CREATE SCHEMA IF NOT EXISTS cockroachlabs;

CREATE TABLE cockroachlabs.movr.users (
);
~~~

The `IF NOT EXISTS` clause of the `CREATE SCHEMA` statement will allow you to execute subsequent statements in the file, even if a schema exists. Before executing the file, however, you need to first [define the columns of the table](#define-the-columns).

### Define columns

Column definitions give structure to a table by separating values of different significance and data type. Column definitions generally take the following form:

~~~
[column_name] [DATA_TYPE] [column_qualification]
~~~

Where `column_name` is the column name, `DATA_TYPE` is the [data type](data-types.html) of the row values in the column, and `column_qualification` is some column qualification, such as a [constraint](#add-additional-constraints). For a simple example, see [below](#column-definition-examples).

#### Column definition best practices

Here are some best practices to follow when defining table columns:

- Review the supported column [data types](data-types.html), and select the appropriate type for the data you plan to store in a column, following the best practices listed on the data type's reference page.
- Use column data types with a fixed size limit, or set a maximum size limit on column data types of variable size (e.g., [`VARBIT(n)`](bit.html#size)). Values exceeding 1MB can lead to [write amplification](https://en.wikipedia.org/wiki/Write_amplification) and cause significant performance degradation.
- Select a primary key, following the best practices for [selecting primary key columns](#select-the-primary-key-columns). After reviewing the primary key best practices, decide if you need to define any dedicated primary key columns.
- Review the best practices for [adding additional constraints](#add-additional-constraints), and decide if you need to define a dedicated primary key column.

#### Column definition examples

In the `dbinit.sql` file, add a few column definitions for the users' names and email addresses to the `users` `CREATE TABLE` statement:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.users (
    first_name STRING,
    last_name STRING,
    email STRING
);
~~~

All of the columns shown above use the [`STRING`](string.html) data type, meaning that any value in any of the columns must be of the data type `STRING`.

Note that CockroachDB supports a number of column data types, including [`DECIMAL`](decimal.html), [`INT`](integer.html), [`TIMESTAMP`](timestamp.html), and [`UUID`](uuid.html), as well as [enumerated data types](#user-defined-types) and [spatial data types](#spatial-data-types). We recommend reviewing [the supported types](data-types.html), and creating columns with data types that corresponds with the types of data that you intend to persist to the cluster from your application.

Let's add another example table to our `movr` schema, with more data types.

As a vehicle-sharing platform, MovR needs to store data about its vehicles. This table should probably include information about the type of vehicle, when it was created, what its status is, and where it is located.

In `dbinit.sql`, under the `CREATE TABLE` statement for `users`, add a definition for a `vehicles` table:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.vehicles (
      id UUID,
      type STRING,
      creation_time TIMESTAMP(TZ),
      status STRING,
      last_location STRING
  );
~~~

This table includes a couple new data types: `UUID`, which we recommend using the `UUID` data type for columns with values that uniquely identify rows (like an "id" column), and `TIMESTAMP(TZ)`, which we recommend for timestamp values.

Note that `type` and `status` column values will likely only be values from a fixed list of possible values. For example, the `type` of a vehicle can only be one of the vehicles supported by the MovR platform (e.g., a `bike`, or a `scooter`). The `status` of a vehicle can only pertain to the availability of a vehicle (e.g., `available` or `unavailable`). For values like this, we recommend using a [user-defined, enumerated type](enum.html).

To create a user-defined type, use a `CREATE TYPE` statement. For example, above the `CREATE TABLE` statement for the `vehicles` table, add the following statements:

{% include copy-clipboard.html %}
~~~
CREATE TYPE vtype AS ENUM ('bike', 'scooter', 'skateboard');
CREATE TYPE vstatus AS ENUM ('available', 'unavailable');
~~~

You can then use the `vtype` and `vstatus` types for `type` and `status` columns:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.vehicles (
      id UUID,
      type vtype,
      creation_time TIMESTAMP(TZ),
      status vstatus,
      last_location STRING
  );
~~~

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `CREATE TYPE` statement, including additional examples, see the [`CREATE TYPE` syntax page](create-tyoe.html). For detailed reference documentation on enumerated data types, including additional examples, see [`ENUM`](enum.html).
{{site.data.alerts.end}}

### Select the primary key columns

A primary key is a column, or set of columns, whose values uniquely identify rows of data. Every table requires a primary key.

When a table is created, CockroachDB creates an index (called the `primary` index) on the primary key column(s). CockroachDB uses this [index](indexes.html) to find rows in a table more efficiently.

Primary keys are defined in `CREATE TABLE` statements with the `PRIMARY KEY` column constraint. The `PRIMARY KEY` constraint requires that all the constrained column(s) contain only unique and non-`NULL` values. To add a single column to a primary key, add the `PRIMARY KEY` keyword to the end of the column definition. To add multiple columns to a primary key (i.e., to create a [composite primary key](https://en.wikipedia.org/wiki/Composite_key)), add a separate `CONSTRAINT "primary" PRIMARY KEY` clause after the column definitions. For examples, see [below](#primary-key-examples).

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `PRIMARY KEY` constraint, including additional examples, see the [`PRIMARY KEY` constraint page](primary-key.html).
{{site.data.alerts.end}}

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

    If you are working with a table that *must* be indexed on sequential keys, use [hash-sharded indexes](hash-sharded-indexes.html). For details about the mechanics and performance improvements of hash-sharded indexes in CockroachDB, see our [Hash Sharded Indexes Unlock Linear Scaling for Sequential Workloads](https://www.cockroachlabs.com/blog/hash-sharded-indexes-unlock-linear-scaling-for-sequential-workloads/) blog post.

#### Primary key examples

To follow a [primary key best practice](#primary-key-best-practices), the `users` and `vehicles` tables in `dbinit.sql` need to explicitly define a primary key.

In the `dbinit.sql` file, add a composite primary key on the `first_name` and `last_name` columns of the `users` table:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.users (
    first_name STRING,
    last_name STRING,
    email STRING,
    CONSTRAINT "primary" PRIMARY KEY (first_name, last_name)
);
~~~

This primary key will uniquely identify rows of user data.

Primary key columns can also be single columns, if those columns are guaranteed to uniquely identify the row, and their values are well-distributed across the cluster.

In the `vehicles` table definition, add a `PRIMARY KEY` constraint on the `id` column:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.vehicles (
      id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
      type STRING,
      creation_time TIMESTAMP(TZ),
      status ENUM,
      last_location STRING
  );
~~~

Note that, in addition to the `PRIMARY KEY` constraint, the `id` column has a `DEFAULT` constraint. This constraint sets a default value for the column to an automatically-generated value, following [`UUID`](uuid.html) best practices. This value is guaranteed to be unique and well-distributed across the cluster. We discuss the `DEFAULT` constraint more [below](#populate-with-default-values).

### Add additional constraints

In addition to the `PRIMARY KEY` constraint, CockroachDB supports a number of other column-level constraints, including `CHECK`, `DEFAULT`, `FOREIGN KEY`, `NOT NULL`, and `UNIQUE`. Using constraints can simplify table queries, improve query performance, and ensure that data remains semantically valid.

To constrain a column, you can add the constraint's clause to the column's definition, as shown in the [`PRIMARY KEY` example above](#primary-key-examples). To constrain more than one column, add the entire constraint's definition after the list of columns in the `CREATE TABLE` statement, also shown in the [`PRIMARY KEY` example above](#primary-key-examples).

For guidance and examples for each constraint, see the sections below.

{{site.data.alerts.callout_success}}
For detailed reference documentation for each supported constraint, see [the constraint's syntax page](constraints.html).
{{site.data.alerts.end}}

#### Populate with default values

To set default values on columns, use the `DEFAULT` constraint. Default values enable you to write queries without the need to specify values for every column.

When combined with [supported functions](functions-and-operators.html), default values can save resources in your application's persistence layer by offloading computation onto CockroachDB. For example, rather than using an additional library for generating unique ID's, you can set a default value to be an automatically-generated `UUID` value with the `gen_random_uuid()` function. Similarly, you could use a default value to populate a `TIMESTAMP` column with the current time of day, using the `now()` function.

For example, in the `vehicles` table definition, you added a `DEFAULT gen_random_uuid()` clause to the `id` column definition. Now, add a default value to the `creation_time` column:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.vehicles (
      id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
      type STRING,
      creation_time TIMESTAMP(TZ) DEFAULT now(),
      status ENUM,
      last_location STRING
  );
~~~

When a row is inserted into the `vehicles` table, CockroachDB generates a random default value for the vehicle `id`, and uses the current timestamp for the vehicle's `creation_time`. Data inserted into the `vehicles` table does not need to include a value for `id` or `creation_time`.

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `DEFAULT` constraint, including additional examples, see [the `DEFAULT` syntax page](default-value.html).
{{site.data.alerts.end}}

#### Reference other tables

To reference values in another table, use a `FOREIGN KEY` constraint. `FOREIGN KEY` constraints enforce [referential integrity](https://en.wikipedia.org/wiki/Referential_integrity), which means that a column can only refer to an existing column.

For example, suppose you want to add a new table for information about the rides that users are taking on vehicles. This table should probably include information about the location and duration of the ride, as well as information about the vehicle used for the ride.

In `dbinit.sql`, under the `CREATE TABLE` statement for `vehicles`, add a definition for a `rides` table, with a foreign key dependency on the `vehicles` table:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.rides (
      id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
      vehicle_id UUID REFERENCES cockroachlabs.movr.vehicles(id),
      start_address STRING,
      end_address STRING,
      start_time TIMESTAMP(TZ) DEFAULT now(),
      end_time TIMESTAMP(TZ)
  )
~~~

The `vehicle_id` column will be identical to the `id` column in the `vehicles` table. Any queries that insert a `vehicle_id` that does not exist in the `id` column of the `vehicles` table will return an error.

{{site.data.alerts.callout_info}}
Foreign key dependencies can significantly impact query performance, as queries involving tables with foreign keys, or tables referenced by foreign keys, require CockroachDB to check two separate tables. We recommend using them sparingly.
{{site.data.alerts.end}}

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `FOREIGN KEY` constraint, including additional examples, see [the `FOREIGN KEY` syntax page](foreign-key.html).
{{site.data.alerts.end}}

#### Prevent duplicates

To prevent duplicate values in a column, use the `UNIQUE` constraint.

For example, suppose that you want to ensure that the email addresses of all users are different, to prevent users from registering for two accounts with the same email address. Add a `UNIQUE` constraint to the `email` column of the `users` table:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.users (
    first_name STRING,
    last_name STRING,
    email STRING UNIQUE,
    CONSTRAINT "primary" PRIMARY KEY (first_name, last_name)
);
~~~

Queries that insert `email` values that already exist in the `users` table will return an error.

{{site.data.alerts.callout_info}}
When you add a `UNIQUE` constraint to a column, CockroachDB creates a secondary index on that column, to help speed up checks on a column value's uniqueness.

Note that the `UNIQUE` constraint is implied by the `PRIMARY KEY` constraint.
{{site.data.alerts.end}}

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `UNIQUE` constraint, including additional examples, see [the `UNIQUE` syntax page](unique.html).
{{site.data.alerts.end}}

#### Prevent `NULL` values

To prevent `NULL` values in a column, use the `NOT NULL` constraint. If you specify a `NOT NULL` constraint, all queries against the table with that constraint must specify a value for that column, or have a default value specified with a `DEFAULT` constraint.

For example, if you require all users of the MovR platform to have an email on file, you can add `NOT NULL` constraint to the `email` column of the `users` table:

{% include copy-clipboard.html %}
~~~
CREATE TABLE cockroachlabs.movr.users (
    first_name STRING,
    last_name STRING,
    email STRING UNIQUE NOT NULL,
    CONSTRAINT "primary" PRIMARY KEY (first_name, last_name)
);
~~~

{{site.data.alerts.callout_info}}
Note that the `NOT NULL` constraint is implied by the `PRIMARY KEY` constraint.
{{site.data.alerts.end}}

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `NOT NULL` constraint, including additional examples, see [the `NOT NULL` syntax page](not-null.html).
{{site.data.alerts.end}}

## See also

- [Create a User-defined Schema](schema-design-schema.html)
- [Add Secondary Indexes](schema-design-indexes.html)
- [CockroachDB naming hierarchy](sql-name-resolution.html#naming-hierarchy)
- [Schema Design Overview](schema-design-overview.html)
- [`CREATE TABLE`](create-table.html)
