{{site.data.alerts.callout_info}}
As a general best practice, we discourage the use of client libraries to execute [database schema changes](online-schema-changes.html). Instead, use a database schema migration tool, or the [CockroachDB SQL client](cockroach-sql.html).

In this guide, we write our database schema object definitions in a SQL file, and then pass the file to the SQL client for execution.
{{site.data.alerts.end}}