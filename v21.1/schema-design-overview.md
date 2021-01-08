---
title: Database Schema Design Overview
summary: An overview of the objects that make up a logical schema
toc: true
---

This page provides an overview of database schemas in CockroachDB.

In later **Design a Database Schema** sections, we provide best practices for designing a database schema that optimizes performance and maximizes storage resources.

## Database Schema Objects

The sections below provide a brief overview of the logical objects that make up a database schema in CockroachDB, for the purpose of orienting application developers.

For detailed documentation on object name resolution, see [Name Resolution](sql-name-resolution.html).

### Databases

Database objects make up the first level of the [CockroachDB naming hierarchy](sql-name-resolution.html#naming-hierarchy). They contain [schemas](#schemas).

{{site.data.alerts.callout_info}}
To avoid confusion with the general term "[database](https://en.wikipedia.org/wiki/Database)", throughout this guide we refer to the logical object as a *database*, to CockroachDB by name, and to a deployment of CockroachDB as a [*cluster*](architecture/overview.html#terms).
{{site.data.alerts.end}}

For guidance on creating databases, see [Create a Database and User-defined Schema](schema-design-database-schema.html).

### Schemas

Schemas make up the second level of the naming hierarchy. Each schema belongs to a single database. Schemas contain [tables](#tables), [indexes](#indexes), and other objects, like [views](#views) and [sequences](#sequences).

By default, all objects in the third level of the naming hierarchy (e.g., tables and indexes) belong to the preloaded `public` schema. Rather than using the `public` schema, we recommend creating your own *user-defined schema*.

{{site.data.alerts.callout_info}}
To avoid confusion with the general term "[schema](https://en.wiktionary.org/wiki/schema)", in this guide we refer to the logical object as a *user-defined schema*, and to the relationship structure of logical objects in a cluster as a *database schema*.
{{site.data.alerts.end}}

For guidance on creating user-defined schemas, see [Create a Database and User-defined Schema](schema-design-database-schema.html).

### Tables

Tables, along with all objects other than [user-defined schemas](#schemas) and [databases](#databases), make up the third and lowest level of the naming hierarchy. Each table can belong to a single user-defined schema. As stated above, all tables belong to the `public` schema by default.

Tables contain rows of data. Each record in a row belongs to a particular column, of a particular data type. Column data can be further qualified with [column-level constraints](column-constraints.html), or computed with [scalar expressions](computed-columns.html).

For guidance on defining tables, see [Tables](schema-design-tables.html).

### Indexes

Indexes, like tables, belong to a single user-defined schema, in the third level of the naming hierarchy.

An index is a sorted copy of the row data in a single table. CockroachDB queries use indexes to find data in a table without having to scan every row of the table.

The two main types of indexes are the `primary` index, an index on the row-identifying [primary key columns](primary-key.html), and the secondary index, an index created on columns of your choice.

For guidance on defining primary keys, see [Create a Table: Select a Primary Key](schema-design-tables.html). For guidance on defining secondary indexes, see [Add a Secondary Index](schema-design-indexes.html).

#### Specialized indexes

CockroachDB supports specialized types of indexes, designed to improve query performance in specific use cases. For guidance on specialized indexes, see [Index a Subset of Rows](partial-indexes.html) and [Index Sequential Keys](hash-sharded-indexes.html).

### Other objects

CockroachDB supports several other objects at the third level of the naming hierarchy, including reusable [views](#views), [sequences](#sequences), and [temporary objects](#temporary-objects).

#### Views

A view is a stored and named selection query.

For guidance on using views, see [Views](views.html).

#### Sequences

Sequences create and store sequential data.

For guidance on using sequences, see [the `CREATE SEQUENCE` syntax page](create-sequence.html).

#### Temporary objects

A temporary object is an object, such as a table, view, or sequence, that is not stored to persistent memory.

For guidance on using temporary objects, see [Temporary Tables](temporary-tables.html).

## Controlling access to objects

CockroachDB supports both user-based and role-based access control. With roles, or with direct assignment, you can grant a [SQL user](authorization#sql-users) the [privileges](authorization,html#privileges) required to view, modify, and delete database schema objects or rows of data.

By default, the user that creates an object is that object's *owner*. [Object owners](authorization.html#object-ownership) have all privileges required to view, modify, or delete that object and the data stored within it.

For more information about ownership, privileges, and authorization, see [Authorization](authorization.html).

## Executing database schema changes

We do not recommend using client drivers or ORM frameworks to execute database schema changes. As a best practice, we recommend creating database schemas and performing database schema changes with one of the following methods:

- Use a compatible database schema migration tool.

    CockroachDB is compatible with most PostgreSQL database schema migration tools, including [Flyway](https://flywaydb.org/) and [Liquibase](https://www.liquibase.com). For a tutorial on performing database schema changes using Liquibase, see our [Liquibase tutorial](liquibase.html). For a tutorial on performing schema changes with Flyway, see our [Flyway tutorial](flyway.html).

- Use the [CockroachDB SQL client](cockroach-sql.html#execute-sql-statements-from-a-file).

    The CockroachDB SQL client allows you to execute commands from the command line, or through the [CockroachDB SQL shell](cockroach-sql.html#sql-shell) interface. From the command line, you can pass a string to the client for execution, or you can pass a `.sql` file populated with SQL commands. From the SQL shell, you can execute SQL commands directly. Throughout the guide, we pass a `.sql` file to the SQL client to perform most database schema changes, as the SQL client is agnostic to application frameworks.



## See also

- [Create a Database and User-defined Schema](schema-design-database-schema.html)
- [CockroachDB naming hierarchy](sql-name-resolution.html#naming-hierarchy)
- [Authorization](authorization.html)
- [Liquibase tutorial](liquibase.html)
- [Flyway](flyway.html)
