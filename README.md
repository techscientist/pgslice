# pgslice

Postgres partitioning as easy as pie. Works great for both new and existing tables, with zero downtime and minimal app changes.

:tangerine: Battle-tested at [Instacart](https://www.instacart.com/opensource)

## Install

pgslice is a command line tool. To install, run:

```sh
gem install pgslice
```

This will give you the `pgslice` command.

## Steps

1. Ensure the table you want to partition has been created. We’ll refer to this as `<table>`.

2. Specify your database credentials

  ```sh
  export PGSLICE_URL=postgres://localhost/myapp_development
  ```

3. Create an intermediate table

  ```sh
  pgslice prep <table> <column> <period>
  ```

  Period can be `day` or `month`.

  This creates a table named `<table>_intermediate` with the appropriate trigger for partitioning.

4. Add partitions

  ```sh
  pgslice add_partitions <table> --intermediate --past 3 --future 3
  ```

  This creates child tables that inherit from the intermediate table.

  Use the `--past` and `--future` options to control the number of partitions.

5. *Optional, for tables with data* - Fill the partitions in batches with data from the original table

  ```sh
  pgslice fill <table>
  ```

  Use the `--batch-size` and `--sleep` options to control the speed.

  To sync data across different databases, check out [pgsync](https://github.com/ankane/pgsync).

6. Swap the intermediate table with the original table

  ```sh
  pgslice swap <table>
  ```

  The original table is renamed `<table>_retired` and the intermediate table is renamed `<table>`.

7. Fill the rest (rows inserted between the first fill and the swap)

  ```sh
  pgslice fill <table> --swapped
  ```

8. Archive and drop the original table

## Adding Partitions

To add partitions, use:

```sh
pgslice add_partitions <table> --future 3
```

Add this as a cron job to create a new partition each day or month.

```sh
# day
0 0 * * * pgslice add_partitions <table> --future 3 --url ...

# month
0 0 1 * * pgslice add_partitions <table> --future 3 --url ...
```

Add a monitor to ensure partitions are being created.

```sql
SELECT 1 FROM
    pg_catalog.pg_class c
INNER JOIN
    pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE
    c.relkind = 'r' AND
    n.nspname = 'public' AND
    c.relname = '<table>_' || to_char(NOW() + INTERVAL '3 days', 'YYYYMMDD')
    -- for months, use to_char(NOW() + INTERVAL '3 months', 'YYYYMM')
```

## Additional Commands

To undo prep (which will delete partitions), use:

```sh
pgslice unprep <table>
```

To undo swap, use:

```sh
pgslice unswap <table>
```

## Sample Output

pgslice prints the SQL commands that were executed on the server. To print without executing, use the `--dry-run` option.

```console
$ pgslice prep visits created_at month
BEGIN;

CREATE TABLE visits_intermediate (LIKE visits INCLUDING ALL);

CREATE FUNCTION visits_insert_trigger()
    RETURNS trigger AS $$
    BEGIN
        EXECUTE 'INSERT INTO visits_' || to_char(NEW.created_at, 'YYYYMM') || ' VALUES ($1.*)' USING NEW;
        RETURN NULL;
    END;
    $$ LANGUAGE plpgsql;

CREATE TRIGGER visits_insert_trigger
    BEFORE INSERT ON visits_intermediate
    FOR EACH ROW EXECUTE PROCEDURE visits_insert_trigger();

COMMIT;
```

```console
$ pgslice add_partitions visits --intermediate --past 1 --future 1
BEGIN;

CREATE TABLE visits_201608
    (CHECK (created_at >= '2016-08-01'::date AND created_at < '2016-09-01'::date))
    INHERITS (visits_intermediate);

ALTER TABLE visits_201608 ADD PRIMARY KEY (id);

CREATE INDEX ON visits_201608 USING btree (user_id);

CREATE TABLE visits_201609
    (CHECK (created_at >= '2016-09-01'::date AND created_at < '2016-10-01'::date))
    INHERITS (visits_intermediate);

ALTER TABLE visits_201609 ADD PRIMARY KEY (id);

CREATE INDEX ON visits_201609 USING btree (user_id);

CREATE TABLE visits_201610
    (CHECK (created_at >= '2016-10-01'::date AND created_at < '2016-11-01'::date))
    INHERITS (visits_intermediate);

ALTER TABLE visits_201610 ADD PRIMARY KEY (id);

CREATE INDEX ON visits_201610 USING btree (user_id);

COMMIT;
```

```console
$ pgslice fill visits
/* 1 of 3 */
INSERT INTO visits_intermediate (id, user_id, ip, created_at)
    SELECT id, user_id, ip, created_at FROM visits
    WHERE id > 0 AND id <= 10000 AND created_at >= '2016-08-01'::date AND created_at < '2016-11-01'::date

/* 2 of 3 */
INSERT INTO visits_intermediate (id, user_id, ip, created_at)
    SELECT id, user_id, ip, created_at FROM visits
    WHERE id > 10000 AND id <= 20000 AND created_at >= '2016-08-01'::date AND created_at < '2016-11-01'::date

/* 3 of 3 */
INSERT INTO visits_intermediate (id, user_id, ip, created_at)
    SELECT id, user_id, ip, created_at FROM visits
    WHERE id > 20000 AND id <= 30000 AND created_at >= '2016-08-01'::date AND created_at < '2016-11-01'::date
```

```console
$ pgslice swap visits
BEGIN;

ALTER TABLE visits RENAME TO visits_retired;

ALTER TABLE visits_intermediate RENAME TO visits;

ALTER SEQUENCE visits_id_seq OWNED BY visits.id;

COMMIT;
```

```console
$ pgslice add_partitions visits --future 2
BEGIN;

CREATE TABLE visits_201611
    (CHECK (created_at >= '2016-11-01'::date AND created_at < '2016-12-01'::date))
    INHERITS (visits);

ALTER TABLE visits_201611 ADD PRIMARY KEY (id);

CREATE INDEX ON visits_201611 USING btree (user_id);

COMMIT;
```

## App Changes

This set up allows you to read and write with the original table name with no knowledge it’s partitioned. However, there are a few things to be aware of.

### Reads

When possible, queries should include the column you partition on to limit the number of partitions the database needs to check.  For instance, if you partition on `created_at`, try to include it in queries:

```sql
SELECT * FROM
    visits
WHERE
    user_id = 123 AND
    -- for performance
    created_at >= '2016-09-01' AND created_at < '2016-09-02'
```

For this to be effective, ensure `constraint_exclusion` is set to `partition` (default value) or `on`.

```sql
SHOW constraint_exclusion;
```

### Writes

If you use `INSERT` statements with a `RETURNING` clause (as frameworks like Rails do), you’ll no longer receive the id of the newly inserted record back. If you need this, you can either:

1. Insert directly into the partition
2. Get the value after the insert with `SELECT CURRVAL('sequence_name')`

## One Off Tasks

You can also use pgslice to reduce the size of a table without partitioning by creating a new table, filling it with a subset of records, and swapping it in.

```sh
pgslice prep <table> --no-partition
pgslice fill <table> --start 1000 # starting primary key
pgslice swap <table>
```

## Upgrading

Run:

```sh
gem install pgslice
```

To use master, run:

```sh
gem install specific_install
gem specific_install ankane/pgslice
```

## Reference

- [PostgreSQL Manual](https://www.postgresql.org/docs/current/static/ddl-partitioning.html)
- [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Table_partitioning)

## TODO

- Command to sync index changes with partitions
- Disable indexing for faster `fill`
- ETA for `fill`

## Related Projects

Also check out:

- [PgHero](https://github.com/ankane/pghero) - A performance dashboard for Postgres
- [pgsync](https://github.com/ankane/pgsync) - Sync Postgres data to your local machine

## Contributing

Everyone is encouraged to help improve this project. Here are a few ways you can help:

- [Report bugs](https://github.com/ankane/pgslice/issues)
- Fix bugs and [submit pull requests](https://github.com/ankane/pgslice/pulls)
- Write, clarify, or fix documentation
- Suggest or add new features
