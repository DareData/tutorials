# Snowflake infrastructure Basics

## Motivation

Any time we have infrastructure, we like to have it as code. This is why we use terraforms and ansible. However, for snowflake, the situation is a bit more complicated. The databases, warehouses, and roles are definitely infrastructure and can be expressed as code. The schemas, tables, and users are more transient than what you'd consider infrastructure.

With all of this in mind, let's take a look at a few of the ways that we can maintain as much as possible using VC-trackable code.

## Terraforms

Our good friend Zuck has managed to get his name unnecessarily on a pretty good set of [terraforms that you can use to manage snowflake infrastructure](https://github.com/chanzuckerberg/terraform-provider-snowflake). You can use these to manage:

1. Databases
1. Schemas (as in the namespace, not the table definition)
1. Roles
1. Users
1. Grants
1. Stages (to ingest from s3)

It is absolutely critical to express all of these as terraforms so that you are able to spin up new environments as necessary. You'll need a testing environment FOR SURE. You'll need a staging environment FOR SURE. And you DEFINITELY do not want to be manually clicking things and specifying permissions for each of these environments. That is a recipe for disaster.

## Tables

You'll notice that tables are not included here. That's for a reason. Tables change a bit too often to really consider them infrastructure. You can keep your table definitions in a directory that contains `.sql` files.

## Migrations

These are tricky. If you need to change things at the terraform level, you can and should just use them. Be sure to TEST TEST TEST before unleashing on prod.

If you need to migrate tables, I don't have a very clean solution. I do have the following process though:

### 1. Create a migration directory

Probably next to your table definitions, you can have a `migrations` directory. Directly inside of this, you can have directories that are in the format `YYYY-MM-DD` so that they can be sorted by a file system. So it might look something like this:

```bash
> tree migrations/
migrations/
├── 2020-01-01
│   └── update-users-table.sql
└── 2020-02-23
    └── change-track-type.sql

2 directories, 2 files
```

And the contents of one of these files could look like this:

```sql
alter table track
add column blah string
;

update table track
set blah = 'some default value'
;
```

And then you'd need to go to the file that contains the source of truth schema and change it to reflect the new changes.

### Exercises

Write terraforms and SQL files to do the following for test, stage, and prod environments:

1. One database called `track_<env>`
1. Two schemas inside of `track_<env>`:
    1. `ingestion`
    1. `public`
1. Have one table in the `public` schema called `tracking` that has two columns:
    1. timestamp (timestamp)
    1. event (string)
1. Have one table in the `ingestion` schema called `tracking` as well that has one column of type variant
1. Have one role called `ingestion_<env>` that has read / write permissions on all tables in the `ingestion` schema
1. Have one role called `etl_<env>` that has full read / write permissions on the `public` schema.
1. Have 3 s3 buckets called `track_<env>`
1. Have 3 different snowflake stages, each of which has read permission to the corresponding bucket