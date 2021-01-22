---
title: Create a User-defined Schema
summary: Create a user-defined schema for your CockroachDB cluster
toc: true
---

This page provides best-practice guidance on creating user-defined schemas, with a simple example based on Cockroach Labs' fictional vehicle-sharing company, [MovR](movr.html).

{{site.data.alerts.callout_success}}
For detailed reference documentation on the `CREATE SCHEMA` statement, including additional examples, see the [`CREATE SCHEMA` syntax page](create-schema.html).
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
- [Create a database](schema-design-database.html)

## Create a user-defined schema

User-defined schemas belong to the second level of the [CockroachDB naming hierarchy](sql-name-resolution.html#naming-hierarchy).

To create a user-defined schema, use a [`CREATE SCHEMA` statement](create-schema.html), following [the user-defined schema best practices](#user-defined-schema-best-practices). See the example we provide [below](#example).

### User-defined schema best practices

Here are some best practices to follow when creating and using user-defined schemas:

- Do not use the preloaded `public` schema. Instead, create a user-defined schema with `CREATE SCHEMA`, and then refer to lower-level objects with fully-qualified names (e.g., `schema_name.table_name`).
- Do not create user-defined schemas in the `defaultdb` database. Instead, use a database you have created. Be sure to change your SQL session's current database before creating a user-defined schema. Schemas can only be created in the current database, and, by default, `defaultdb` is the current database.
- When you create a user-defined schema, take note of the owner. You can specify the owner with the `AUTHORIZATION` keyword. If `AUTHORIZATION` is not specified, the owner will be the creator. We recommend creating user-defined schemas as the user that will create the lower-level objects in the database schema.

### Example

{% include {{page.version.version}}/sql/dev-schema-changes.md %}

For the examples in the rest of the **Design a Database Schema** sections, we'll add all database schema change statements to a `.sql` file, and then pass the file to the [SQL client](cockroach-sql.html) for execution as the `max` user.

Create an empty file with the `.sql` file extension at the end of the filename. For example:

{% include copy-clipboard.html %}
~~~ shell
$ touch dbinit.sql
~~~

Now, open `dbinit.sql` in a text editor of your choice, and add a `CREATE SCHEMA` statement:

{% include copy-clipboard.html %}
~~~
CREATE SCHEMA IF NOT EXISTS movr;
~~~

This statement will create a user-defined schema named `movr` in the current database, if one does not already exist.

To execute the statements in the `dbinit.sql` file, run the following command:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--certs-dir=[certs-directory] \
--user=max \
--database=cockroachlabs
< dbinit.sql
~~~

The SQL client will execute any statements in `dbinit.sql`, with `cockroachlabs` as the current database and `max` as the user.

After you have executed the statements, you can see the `cockroachlabs` database and the `movr` user-defined schema in the [CockroachDB SQL shell](cockroach-sql.html#sql-shell). To open the SQL shell with `cockroachlabs` as the current database, run the following command:

{% include copy-clipboard.html %}
~~~ shell
$ cockroach sql \
--certs-dir=[certs-directory] \
--user=max \
--database=cockroachlabs
~~~

To view the user-defined schemas in the cluster, issue a [`SHOW SCHEMAS`](show-schemas.html) statement:

{% include copy-clipboard.html %}
~~~ sql
> SHOW SCHEMAS;
~~~

~~~
     schema_name
----------------------
  crdb_internal
  information_schema
  movr
  pg_catalog
  pg_extension
  public
(6 rows)
~~~

You're now ready to start adding tables to the user-defined schema. For guidance on creating tables, see at [Create a Table](schema-design-table.html).

## See also

- [Create a Database](schema-design-database.html)
- [Create a Table](schema-design-table.html)
- [CockroachDB naming hierarchy](sql-name-resolution.html#naming-hierarchy)
- [Schema Design Overview](schema-design-overview.html)
- [`CREATE SCHEMA`](create-schema.html)
