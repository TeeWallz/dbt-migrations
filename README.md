# dbt-migrations
A dbt plugin to support database migrations.

## Background

*In feedback phase, very unstable!*

This plugin introduces the concept of database migrations, which on the surface is counter to the dbt "everything is a select statement" mantra.

It's important to note that this plugin is specifically *not* for creating tables, views or any other type of relation. Instead, it is to manage the many other types of objects present in modern analytics databases (stages, file formats, pipes, tasks, streams, etc).

Currently, ensuring the migrations run in order is a little tricky, and it's necessary to create model dependancies by convention (just an incrementing number). [In future](https://github.com/fishtown-analytics/dbt/issues/1212) it maybe be possible to support nicely named migration files with semantic versioning in their names.,

## How it works

This plugin includes:
1) A new materialization named "migration" which runs the model SQL verbatim, but only if it hasn't been ran before. It creates and maintains a table named "change_history" under a custom schema named "migrations" to achieve this.
2) A macro named "enfore_migration_dependancy" which, when added to a model, ensures it depends on another model whose name is (numerically) one less than itself. This ensures that migrations are ran in the correct order.

## Usage instructions

1. Add to packages.yml in your dbt project root:
```
packages:    
  - git: "https://github.com/TeeWallz/dbt-migrations.git"
    revision: master
```

2. Run `dbt deps`

3. Add "migrations" to your source-paths array in project.yml

4. Create the migrations table
```sql
dbt run  --select change_history
```

5. Create migration files like so:

migrations/1_my_csv_format.sql
```
{{ config(materialized='migration') }}
{{ dbt_migrations.enforce_migration_dependancy() }}

create or replace file format my_csv_format
  type = csv
  field_delimiter = '|'
  skip_header = 1
  null_if = ('NULL', 'null')
  empty_field_as_null = true
  compression = gzip;
```

migrations/2_my_int_stage.sql
```
{{ config(materialized='migration') }}
{{ dbt_migrations.enforce_migration_dependancy() }}

create or replace stage my_int_stage
  file_format = my_csv_format;

```

6. `dbt run`
