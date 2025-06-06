---
title: Paginate Results
summary: Paginate results from queries against your cluster
toc: true
docs_area: develop
---

To iterate through a table one "page" of results at a time (also known as pagination) there are several options:

- [`LIMIT` / `OFFSET`]({% link {{ page.version.version }}/limit-offset.md %}) pagination (slow, not recommended)
- Keyset pagination (**fast, recommended**)
- [Cursors]({% link {{ page.version.version }}/cursors.md %}) (it depends, see [Differences between keyset pagination and cursors](#differences-between-keyset-pagination-and-cursors))

## Keyset pagination

Keyset pagination (also known as the "seek method") is used to fetch a subset of records from a table quickly. It does this by restricting the set of records returned with a combination of `WHERE` and [`LIMIT`]({% link {{ page.version.version }}/limit-offset.md %}) clauses. To get the next page, you check the value of the column in the `WHERE` clause against the last row returned in the previous page of results.

The general pattern for keyset pagination queries is:

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM t AS OF SYSTEM TIME ${time}
  WHERE key > ${value}
  ORDER BY key
  LIMIT ${amount}
~~~

This is faster than using `LIMIT`/`OFFSET` because, instead of doing a full table scan up to the value of the `OFFSET`, a keyset pagination query looks at a fixed-size set of records for each iteration. This can be done quickly provided that the key used in the `WHERE` clause to implement the pagination is [indexed]({% link {{ page.version.version }}/indexes.md %}#best-practices) and [unique]({% link {{ page.version.version }}/unique.md %}). A [primary key]({% link {{ page.version.version }}/primary-key.md %}) meets both of these criteria.

## Examples

The examples in this section use the [employees data set](https://github.com/datacharmer/test_db). 

You will need to write a [`CREATE TABLE`]({% link {{ page.version.version }}/create-table.md %}) statement that matches the schema of the table data you're importing.

For example, to import the data from `employees.csv` into an `employees` table, issue the following statement to create the table:

{% include_cached copy-clipboard.html %}
~~~sql
CREATE TABLE employees (
  emp_no INT PRIMARY KEY,
  birth_date DATE NOT NULL,
  first_name STRING NOT NULL,
  last_name STRING NOT NULL,
  gender STRING NOT NULL,
  hire_date DATE NOT NULL
      );
~~~

Next, use [`IMPORT INTO`]({% link {{ page.version.version }}/import-into.md %}) to import the data into the new table:

{% include_cached copy-clipboard.html %}
~~~sql
IMPORT INTO employees (emp_no, birth_date, first_name, last_name, gender, hire_date)
     CSV DATA (
       'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/csv/employees.csv.gz'
     );
~~~

~~~
       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes   
--------------------+-----------+--------------------+--------+---------------+----------------+----------
 381866942129111041 | succeeded |                  1 | 300024 |             0 |              0 | 13258389
(1 row)
~~~

To get the first page of results using keyset pagination, run the statement below.

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM employees AS OF SYSTEM TIME '-1m' WHERE emp_no > 10000 ORDER BY emp_no LIMIT 25;
~~~

~~~
emp_no |        birth_date         | first_name |  last_name  | gender |         hire_date
---------+---------------------------+------------+-------------+--------+----------------------------
 10001 | 1953-09-02 00:00:00+00:00 | Georgi     | Facello     | M      | 1986-06-26 00:00:00+00:00
 10002 | 1964-06-02 00:00:00+00:00 | Bezalel    | Simmel      | F      | 1985-11-21 00:00:00+00:00
 10003 | 1959-12-03 00:00:00+00:00 | Parto      | Bamford     | M      | 1986-08-28 00:00:00+00:00
 10004 | 1954-05-01 00:00:00+00:00 | Chirstian  | Koblick     | M      | 1986-12-01 00:00:00+00:00
 ...
(25 rows)

Time: 4ms total (execution 3ms / network 0ms)
~~~

{{site.data.alerts.callout_success}}
When writing your own queries of this type, use a known minimum value for the key's data type. If you do not know what the minimum value of the key is, you can use `SELECT min(key) FROM table`.
{{site.data.alerts.end}}

{{site.data.alerts.callout_info}}
We use [`AS OF SYSTEM TIME`]({% link {{ page.version.version }}/as-of-system-time.md %}) in these examples to ensure that we are operating on a consistent snapshot of the database as of the specified timestamp. This reduces the chance that there will be any concurrent updates to the data the query is accessing, and thus no missing or duplicated rows during the pagination. It also reduces the risk of [client-side transaction retries]({% link {{ page.version.version }}/transaction-retry-error-reference.md %}#client-side-retry-handling) due to concurrent data access. The value of `-1m` passed to `AS OF SYSTEM TIME` may need to be updated depending on your application's data access patterns.
{{site.data.alerts.end}}

To get the second page of results, run:

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM employees AS OF SYSTEM TIME '-1m' WHERE emp_no > 10025 ORDER BY emp_no LIMIT 25;
~~~

~~~
  emp_no |        birth_date         | first_name | last_name  | gender |         hire_date
---------+---------------------------+------------+------------+--------+----------------------------
   10026 | 1953-04-03 00:00:00+00:00 | Yongqiao   | Berztiss   | M      | 1995-03-20 00:00:00+00:00
   10027 | 1962-07-10 00:00:00+00:00 | Divier     | Reistad    | F      | 1989-07-07 00:00:00+00:00
   10028 | 1963-11-26 00:00:00+00:00 | Domenick   | Tempesti   | M      | 1991-10-22 00:00:00+00:00
   10029 | 1956-12-13 00:00:00+00:00 | Otmar      | Herbst     | M      | 1985-11-20 00:00:00+00:00
...
(25 rows)

Time: 2ms total (execution 1ms / network 1ms)
~~~

To get an arbitrary page of results showing employees whose IDs (`emp_no`) are in a much higher range, run the following query. Note that it takes about the same amount of time to run as the previous queries.

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM employees AS OF SYSTEM TIME '-1m' WHERE emp_no > 300025 ORDER BY emp_no LIMIT 25;
~~~

~~~
  emp_no |        birth_date         | first_name |  last_name   | gender |         hire_date
---------+---------------------------+------------+--------------+--------+----------------------------
  400000 | 1963-11-29 00:00:00+00:00 | Mitsuyuki  | Reinhart     | M      | 1985-08-27 00:00:00+00:00
  400001 | 1962-06-02 00:00:00+00:00 | Rosalie    | Chinin       | M      | 1986-11-28 00:00:00+00:00
  400002 | 1964-08-16 00:00:00+00:00 | Quingbo    | Birnbaum     | F      | 1986-04-23 00:00:00+00:00
  400003 | 1958-04-30 00:00:00+00:00 | Jianwen    | Sidhu        | M      | 1986-02-01 00:00:00+00:00
  400004 | 1958-04-30 00:00:00+00:00 | Sedat      | Suppi        | M      | 1995-12-18 00:00:00+00:00
....
(25 rows)

Time: 2ms total (execution 1ms / network 0ms)
~~~

Compare the execution speed of the previous keyset pagination queries with the query below that uses `LIMIT` / `OFFSET` to get the same page of results:

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM employees AS OF SYSTEM TIME '-1m' LIMIT 25 OFFSET 200024;
~~~

~~~
  emp_no |        birth_date         | first_name |  last_name   | gender |         hire_date
---------+---------------------------+------------+--------------+--------+----------------------------
  400000 | 1963-11-29 00:00:00+00:00 | Mitsuyuki  | Reinhart     | M      | 1985-08-27 00:00:00+00:00
  400001 | 1962-06-02 00:00:00+00:00 | Rosalie    | Chinin       | M      | 1986-11-28 00:00:00+00:00
  400002 | 1964-08-16 00:00:00+00:00 | Quingbo    | Birnbaum     | F      | 1986-04-23 00:00:00+00:00
  400003 | 1958-04-30 00:00:00+00:00 | Jianwen    | Sidhu        | M      | 1986-02-01 00:00:00+00:00
...
(25 rows)

Time: 158ms total (execution 156ms / network 1ms)
~~~

The query using `LIMIT`/`OFFSET` for pagination is almost 100 times slower. To see why, let's use [`EXPLAIN`]({% link {{ page.version.version }}/explain.md %}).

{% include_cached copy-clipboard.html %}
~~~ sql
EXPLAIN SELECT * FROM employees LIMIT 25 OFFSET 200024;
~~~

~~~
                                          info
----------------------------------------------------------------------------------------
  distribution: full
  vectorized: true

  • limit
  │ estimated row count: 25
  │ offset: 200024
  │
  └── • scan
        estimated row count: 200,049 (67% of the table; stats collected 5 minutes ago)
        table: employees@idx_17110_primary
        spans: LIMITED SCAN
        limit: 200049
(12 rows)

Time: 4ms total (execution 3ms / network 0ms)
~~~

The culprit is this: because we used `LIMIT`/`OFFSET`, we are performing a limited scan of the entire table (see `spans: LIMITED SCAN` above) from the first record all the way up to the value of the offset. In other words, we are iterating over a big array of rows from 1 to *n*, where *n* is 200049. The `estimated row count` row shows this.

Meanwhile, the keyset pagination queries are looking at a much smaller range of table spans, which is much faster (see `spans: [/300026 - ]` and `limit: 25` below). Because [there is an index on every column in the `WHERE` clause]({% link {{ page.version.version }}/indexes.md %}#best-practices), these queries are doing an index lookup to jump to the start of the page of results, and then getting an additional 25 rows from there. This is much faster.

{% include_cached copy-clipboard.html %}
~~~ sql
EXPLAIN SELECT * FROM employees WHERE emp_no > 300025 ORDER BY emp_no LIMIT 25;
~~~

~~~
                                       info
----------------------------------------------------------------------------------
  distribution: local
  vectorized: true

  • scan
    estimated row count: 25 (<0.01% of the table; stats collected 6 minutes ago)
    table: employees@idx_17110_primary
    spans: [/300026 - ]
    limit: 25
(8 rows)

Time: 1ms total (execution 1ms / network 0ms)
~~~

As shown by the `estimated row count` row, this query scans only 25 rows, far fewer than the 200049 scanned by the `LIMIT`/`OFFSET` query.

{{site.data.alerts.callout_danger}}
Using a sequential (i.e., non-[UUID]({% link {{ page.version.version }}/uuid.md %})) primary key creates hotspots in the database for write-heavy workloads, since concurrent [`INSERT`]({% link {{ page.version.version }}/insert.md %})s to the table will attempt to write to the same (or nearby) underlying [ranges]({% link {{ page.version.version }}/architecture/overview.md %}#architecture-range). This can be mitigated by designing your schema with [multi-column primary keys which include a monotonically increasing column]({% link {{ page.version.version }}/performance-best-practices-overview.md %}#use-multi-column-primary-keys).
{{site.data.alerts.end}}

## Differences between keyset pagination and cursors

{% include {{page.version.version}}/sql/cursors-vs-keyset-pagination.md %}

## See also

- [Cursors]({% link {{ page.version.version }}/cursors.md %})
- [`LIMIT` / `OFFSET`]({% link {{ page.version.version }}/limit-offset.md %}) pagination (slow, not recommended)
