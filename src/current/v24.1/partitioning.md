---
title: Table Partitioning
summary: Partitioning gives you row-level control of how and where your data is stored.
toc: true
docs_area: deploy
---

CockroachDB allows you to define table partitions, giving you row-level control of how and where your data is stored. Partitioning enables you to reduce latencies and costs and can assist in meeting regulatory requirements for your data.



## Why use table partitioning

{% include {{page.version.version}}/sql/use-multiregion-instead-of-partitioning.md %}

Table partitioning helps you reduce latency and cost:

- **Geo-partitioning** allows you to keep user data close to the user, which reduces the distance that the data needs to travel, thereby **reducing latency**. To geo-partition a table, define location-based partitions while creating a table, create location-specific [zone configurations]({% link {{ page.version.version }}/configure-replication-zones.md %}), and apply the zone configurations to the corresponding partitions.
- **Archival-partitioning** allows you to store infrequently-accessed data on slower and cheaper storage, thereby **reducing costs**. To archival-partition a table, define frequency-based partitions while creating a table, create frequency-specific [zone configurations]({% link {{ page.version.version }}/configure-replication-zones.md %}) with appropriate storage devices constraints, and apply the zone configurations to the corresponding partitions.

## How it works

Table partitioning involves a combination of CockroachDB features:

- [Node attributes](#node-attributes)
- [Table creation](#table-creation)
- [Replication zones](#replication-zones)

### Node attributes

To store partitions in specific locations (e.g., geo-partitioning), or on machines with specific attributes (e.g., archival-partitioning), the nodes of your cluster must be [started]({% link {{ page.version.version }}/cockroach-start.md %}) with the relevant flags:

- Use the `--locality` flag to assign key-value pairs that describe the location of a node, for example, `--locality=region=east,az=us-east-1`.
- Use the `--attrs` flag to specify node capability, which might include specialized hardware or number of cores, for example, `--attrs=ram:64gb`.
- Use the `attrs` field of the `--store` flag to specify disk type or capability, for example,`--store=path=/mnt/ssd01,attrs=ssd`.

For more details about these flags, see the [`cockroach start`]({% link {{ page.version.version }}/cockroach-start.md %}) documentation.

### Table creation

You can define partitions and subpartitions over one or more columns of a table. During [table creation]({% link {{ page.version.version }}/create-table.md %}), you declare which values belong to each partition in one of two ways:

- **List partitioning**: Enumerate all possible values for each partition. List partitioning is a good choice when the number of possible values is small. List partitioning is well-suited for geo-partitioning.
- **Range partitioning**: Specify a contiguous range of values for each partition by specifying lower and upper bounds. Range partitioning is a good choice when the number of possible values is too large to explicitly list out. Range partitioning is well-suited for archival-partitioning.

#### Partition by list

[`PARTITION BY LIST`]({% link {{ page.version.version }}/alter-table.md %}#partition-by) lets you map one or more tuples to a partition.

To partition a table by list, use the [`PARTITION BY LIST`]({% link {{ page.version.version }}/alter-table.md %}#partition-by) syntax while creating the table. While defining a list partition, you can also set the `DEFAULT` partition that acts as a catch-all if none of the rows match the requirements for the defined partitions.

See [Partition by List](#define-table-partitions-by-list) example below for more details.

#### Partition by range

[`PARTITION BY RANGE`]({% link {{ page.version.version }}/alter-table.md %}#partition-by) lets you map ranges of tuples to a partition.

To define a table partition by range, use the [`PARTITION BY RANGE`]({% link {{ page.version.version }}/alter-table.md %}#partition-by) syntax while creating the table.  While defining a range partition, you can use CockroachDB-defined `MINVALUE` and `MAXVALUE` parameters to define the lower and upper bounds of the ranges respectively.

{{site.data.alerts.callout_info}}The lower bound of a range partition is inclusive, while the upper bound is exclusive. For range partitions, <code>NULL</code> is considered less than any other data, which is consistent with our key encoding ordering and <code>ORDER BY</code> behavior.{{site.data.alerts.end}}

Partition values can be any SQL expression, but it’s only evaluated once. If you create a partition with value `< (now() - '1d')` on 2017-01-30, it would be contain all values less than 2017-01-29. It would not update the next day, it would continue to contain values less than 2017-01-29.

See [Partition by Range](#define-table-partitions-by-range) example below for more details.

#### Partition using primary key

The primary key required for partitioning is different from the conventional primary key. To define the primary key for partitioning, prefix the unique identifier(s) in the primary key with all columns you want to partition and subpartition the table on, in the order in which you want to nest your subpartitions.

For instance, consider the database of a global online learning portal that has a table for students of all the courses across the world. If you want to geo-partition the table based on the countries of the students, then the primary key needs to be defined as:

{% include_cached copy-clipboard.html %}
~~~ sql
> CREATE TABLE students (
    id INT DEFAULT unique_rowid(),
    name STRING,
    email STRING,
    country STRING,
    expected_graduation_date DATE,
    PRIMARY KEY (country, id));
~~~

##### Primary key considerations

The order in which the columns are defined in the primary key is important. The partitions and subpartitions need to follow that order. In the example of the online learning portal, if you define the primary key as `(country, expected_graduation_date, id)`, the primary partition is by `country`, and then subpartition is by `expected_graduation_date`. You cannot skip `country` and partition by `expected_graduation_date`.

#### Partition using a secondary index

The primary key discussed in the preceding section has two drawbacks:

- It does not enforce that the identifier column is globally unique.
- It does not provide fast lookups on the identifier.

To ensure uniqueness or fast lookups, create a [secondary index]({% link {{ page.version.version }}/indexes.md %}) on the identifier.

Indexes are not required to be partitioned, but creating a non-partitioned index on a partitioned table may not perform well.

When you create a non-partitioned index on a partitioned table, CockroachDB sends a [`NOTICE` message](https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html) to the client stating that creating a non-partitioned index on a partitioned table may not perform well.

#### Partition using foreign key reference

If a partitioned table contains a [foreign key reference]({% link {{ page.version.version }}/foreign-key.md %}) to a non-partitioned table, the secondary index created automatically for the foreign key reference will not be partitioned. This can impact performance when querying against the partitioned table, as the data may exist in a distant node.

To minimize potential latency issues, configure the non-partitioned table to be a [`GLOBAL` table]({% link {{ page.version.version }}/global-tables.md %}).

### Replication zones

On their own, partitions are inert and simply apply a label to the rows of the table that satisfy the criteria of the defined partitions. Applying functionality to a partition requires creating and applying [replication zone]({% link {{ page.version.version }}/configure-replication-zones.md %}) to the corresponding partitions.

CockroachDB uses the most granular zone config available. Zone configs that target a partition are considered more granular than those that target a table or index, which in turn are considered more granular than those that target a database.

For more information about how zone config inheritance works, see [Replication Controls]({% link {{ page.version.version }}/configure-replication-zones.md %}#level-priorities).

{% include {{page.version.version}}/sql/querying-partitions.md %}

## Examples

### Define table partitions by list

Consider a global online learning portal, RoachLearn, that has a database containing a table of students across the world. Suppose we have three availability zone: one in the United States, one in Germany, and another in Australia. To reduce latency, we want to keep the students' data closer to their locations:

- We want to keep the data of the students located in the United States and Canada in the United States availability zone.
- We want to keep the data of students located in Germany and Switzerland in the German availability zone.
- We want to keep the data of students located in Australia and New Zealand in the Australian availability zone.

#### Step 1. Identify the partitioning method

We want to geo-partition the table to keep the students' data closer to their locations. We can achieve this by partitioning on `country` and using the `PARTITION BY LIST` syntax.

#### Step 2. Start each node with its availability zone location specified in the `--locality` flag

1. In separate terminal windows, start three nodes in the US availability zone:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=us \
    --listen-addr=localhost:26257 \
    --http-addr=localhost:8080 \
    --store=node1 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=us \
    --listen-addr=localhost:26258 \
    --http-addr=localhost:8081 \
    --store=node2 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=us \
    --listen-addr=localhost:26259 \
    --http-addr=localhost:8082 \
    --store=node3 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

1. In a new terminal window, initialize the cluster:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach init \
    --insecure \
    --host=localhost:26257
    ~~~

1. In new terminal windows, start three more nodes in the German availability zone:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=de \
    --listen-addr=localhost:26260 \
    --http-addr=localhost:8083 \
    --store=node4 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=de \
    --listen-addr=localhost:26261 \
    --http-addr=localhost:8084 \
    --store=node5 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=de \
    --listen-addr=localhost:26262 \
    --http-addr=localhost:8085 \
    --store=node6 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

1. In separate terminal windows, start three nodes in the Australian availability zone:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=aus \
    --listen-addr=localhost:26263 \
    --http-addr=localhost:8086 \
    --store=node7 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=aus \
    --listen-addr=localhost:26264 \
    --http-addr=localhost:8087 \
    --store=node8 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start \
    --insecure \
    --listen-addr=localhost \
    --locality=region=aus \
    --listen-addr=localhost:26265 \
    --http-addr=localhost:8088 \
    --store=node9 \
    --join=localhost:26257,localhost:26258,localhost:26259
    ~~~


#### Step 3. Request and set a trial Enterprise license

See [Set the Trial or Enterprise License Key]({% link {{ page.version.version }}/licensing-faqs.md %}#set-a-license).

#### Step 4. Create the `roachlearn` database and `students` table

1. Open the CockroachDB SQL shell:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach sql --insecure --host=localhost:26257
    ~~~

1. Create the database and set it as current:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > CREATE DATABASE roachlearn;
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > SET DATABASE = roachlearn;
    ~~~

1. Create the table with the appropriate partitions:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE students (
        id INT DEFAULT unique_rowid(),
        name STRING,
        email STRING,
        country STRING,
        expected_graduation_date DATE,
        PRIMARY KEY (country, id))
        PARTITION BY LIST (country) (
          PARTITION north_america VALUES IN ('CA','US'),
          PARTITION europe VALUES IN ('DE', 'CH'),
          PARTITION australia VALUES IN ('AU','NZ'),
          PARTITION DEFAULT VALUES IN (default)
        );
    ~~~

    Alternatively, you can create and partition the table as separate steps:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > CREATE TABLE students (
        id INT DEFAULT unique_rowid(),
        name STRING,
        email STRING,
        country STRING,
        expected_graduation_date DATE,
        PRIMARY KEY (country, id));
    ~~~

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > ALTER TABLE students
      PARTITION BY LIST (country) (
        PARTITION north_america VALUES IN ('CA','US'),
        PARTITION europe VALUES IN ('DE', 'CH'),
        PARTITION australia VALUES IN ('AU','NZ'),
        PARTITION DEFAULT VALUES IN (default)
      );
    ~~~

#### Step 5. Create and apply corresponding replication zones

To create replication zone and apply them to corresponding partitions, use the [`ALTER PARTITION ... CONFIGURE ZONE`]({% link {{ page.version.version }}/alter-partition.md %}#create-a-replication-zone-for-a-partition) statement:

1. Create a replication zone for the `north_america` partition and constrain its data to the US availability zone:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > ALTER PARTITION north_america OF TABLE students
        CONFIGURE ZONE USING constraints='[+region=us]';
    ~~~

1. Create a replication zone for the `europe` partition and constrain its data to the German availability zone:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > ALTER PARTITION europe OF TABLE students
        CONFIGURE ZONE USING constraints='[+region=de]';
    ~~~

1. Create a replication zone for the `australia` partition and constrain its data to the Australian availability zone:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > ALTER PARTITION australia OF TABLE students
        CONFIGURE ZONE USING constraints='[+region=aus]';
    ~~~

1. After creating these replication zones, you can view them using the [`SHOW ZONE CONFIGURATION`]({% link {{ page.version.version }}/show-zone-configurations.md %}) statement:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > SHOW ZONE CONFIGURATION FROM PARTITION north_america OF TABLE students;
    ~~~

    ~~~
                   target                   |                            raw_config_sql
+-------------------------------------------+----------------------------------------------------------------------+
  PARTITION north_america OF TABLE students | ALTER PARTITION north_america OF TABLE students CONFIGURE ZONE USING
                                            |     range_min_bytes = 134217728,
                                            |     range_max_bytes = 536870912,
                                            |     gc.ttlseconds = 90000,
                                            |     num_replicas = 3,
                                            |     constraints = '[+region=us]',
                                            |     lease_preferences = '[]'
(1 row)
    ~~~


#### Step 6. Check replica distribution

{% include_cached copy-clipboard.html %}
~~~ sql
> WITH x AS (SHOW RANGES FROM TABLE roachlearn.students) SELECT * FROM x WHERE "start_key" IS NOT NULL AND "start_key" NOT LIKE '%Prefix%';
~~~

You should see the following output:

~~~
  start_key |     end_key     | range_id | replicas | lease_holder
+-----------+-----------------+----------+----------+--------------+
  /"AU"     | /"AU"/PrefixEnd |       35 | {7,8,9}  |            8
  /"CA"     | /"CA"/PrefixEnd |       31 | {1,2,3}  |            1
  /"CH"     | /"CH"/PrefixEnd |       51 | {4,5,6}  |            5
  /"DE"     | /"DE"/PrefixEnd |       53 | {4,5,6}  |            5
  /"NZ"     | /"NZ"/PrefixEnd |       55 | {7,8,9}  |            8
  /"US"     | /"US"/PrefixEnd |       33 | {1,2,3}  |            1
(6 rows)
~~~

For reference, here's how the nodes map to zones:

Node IDs | Zone
---------|-----
1-3 | `north_america`
4-6 | `europe`
7-9 | `australia`

We can see that, after partitioning, the replicas for `US` and `CA`-based students are located on nodes 1-3 in `north_america`, the replicas for `DE` and `CH`-based students are located on nodes 4-6 in `europe`, and the replicas for `AU` and `NZ`-based students are located on nodes 7-9 in `australia`.

### Show partitions and zone constraints

To retrieve table partitions, you can use the [`SHOW PARTITIONS`]({% link {{ page.version.version }}/show-partitions.md %}) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW PARTITIONS FROM TABLE students;
~~~

~~~
  database_name | table_name | partition_name | parent_partition | column_names |    index_name    | partition_value | zone_constraints
+---------------+------------+----------------+------------------+--------------+------------------+-----------------+------------------+
  roachlearn    | students   | north_america  | NULL             | country      | students@primary | ('CA'), ('US')  | [+region=us]
  roachlearn    | students   | europe         | NULL             | country      | students@primary | ('DE'), ('CH')  | [+region=de]
  roachlearn    | students   | australia      | NULL             | country      | students@primary | ('AU'), ('NZ')  | [+region=aus]
  roachlearn    | students   | default        | NULL             | country      | students@primary | (DEFAULT)       | NULL
(4 rows)
~~~

You can also view partitions by [database]({% link {{ page.version.version }}/show-partitions.md %}#show-partitions-by-database) and [index]({% link {{ page.version.version }}/show-partitions.md %}#show-partitions-by-index).

{% include {{page.version.version}}/sql/crdb-internal-partitions.md %}

### Define table partitions by range

Suppose we want to store the data of current students on fast and expensive storage devices (e.g., SSD) and store the data of the graduated students on slower, cheaper storage devices (e.g., HDD).

#### Step 1. Identify the partitioning method

We want to archival-partition the table to keep newer data on faster devices and older data on slower devices. We can achieve this by partitioning the table by date and using the `PARTITION BY RANGE` syntax.

#### Step 2. Set the Enterprise license

To set the Enterprise license, see [Set the Trial or Enterprise License Key]({% link {{ page.version.version }}/licensing-faqs.md %}#set-a-license).

#### Step 3. Start each node with the appropriate storage device specified in the `--store` flag

1. Start the first node:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start --insecure \
    --store=path=/mnt/1,attrs=ssd \
    --advertise-addr=<node1 hostname> \
    --join=<node1 hostname>,<node2 hostname>
    ~~~

1. Start the second node:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach start --insecure \
    --store=path=/mnt/2,attrs=hdd \
    --advertise-addr=<node2 hostname> \
    --join=<node1 hostname>,<node2 hostname>
    ~~~

1. Initialize the cluster:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    $ cockroach init \
    --insecure \
    --host=<address of any node>
    ~~~

#### Step 4. Create a table with the appropriate partitions

{% include_cached copy-clipboard.html %}
~~~ sql
> CREATE TABLE students_by_range (
   id INT DEFAULT unique_rowid(),
   name STRING,
   email STRING,
   country STRING,
   expected_graduation_date DATE,
   PRIMARY KEY (expected_graduation_date, id))
   PARTITION BY RANGE (expected_graduation_date)
      (PARTITION graduated VALUES FROM (MINVALUE) TO ('2017-08-15'),
      PARTITION current VALUES FROM ('2017-08-15') TO (MAXVALUE));
~~~

#### Step 5. Create and apply corresponding zone configurations

To create zone configurations and apply them to corresponding partitions, use the [`ALTER PARTITION ... CONFIGURE ZONE`]({% link {{ page.version.version }}/alter-partition.md %}#create-a-replication-zone-for-a-partition) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER PARTITION current OF TABLE students_by_range
    CONFIGURE ZONE USING constraints='[+ssd]';
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER PARTITION graduated OF TABLE students_by_range
    CONFIGURE ZONE USING constraints='[+hdd]';
~~~

#### Step 6. Check replica distribution

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW RANGES FROM TABLE students_by_range;
~~~

You should see the following output:

~~~
  start_key | end_key | range_id | replicas | lease_holder | locality
+-----------+---------+----------+----------+--------------+-----------+
  NULL      | /17393  |       52 | {2,5,8}  |            5 | region=de
  /17393    | NULL    |       53 | {6,7,10} |            6 | region=de
(2 rows)
~~~

### Define subpartitions on a table

A list partition can itself be partitioned, forming a subpartition. There is no limit on the number of levels of subpartitioning; that is, list partitions can be infinitely nested.

{{site.data.alerts.callout_info}}Range partitions cannot be subpartitioned.{{site.data.alerts.end}}

Going back to RoachLearn's scenario, suppose we want to do all of the following:

- Keep the students' data close to their location.
- Store the current students' data on faster storage devices.
- Store the graduated students' data on slower, cheaper storage devices (example: HDD).

#### Step 1. Identify the Partitioning method

We want to geo-partition as well as archival-partition the table. We can achieve this by partitioning the table first by location and then by date.

#### Step 2. Start each node with the appropriate storage device specified in the `--store` flag

Start a node in the US availability zone:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--advertise-addr=<node1 hostname> \
--locality=az=us1 \
--store=path=/mnt/1,attrs=ssd \
--store=path=/mnt/2,attrs=hdd \
--join=<node1 hostname>,<node2 hostname>
~~~

Start a node in the AUS availability zone:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach start \
--insecure \
--advertise-addr=<node2 hostname> \
--locality=az=aus1 \
--store=path=/mnt/3,attrs=ssd \
--store=path=/mnt/4,attrs=hdd \
--join=<node1 hostname>,<node2 hostname>
~~~

Initialize the cluster:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach init --insecure --host=<address of any node>
~~~

#### Step 3. Set the Enterprise license

To set the Enterprise license, see [Set the Trial or Enterprise License Key]({% link {{ page.version.version }}/licensing-faqs.md %}#set-a-license).

#### Step 4. Create a table with the appropriate partitions

{% include_cached copy-clipboard.html %}
~~~ sql
> CREATE TABLE students (
    id INT DEFAULT unique_rowid(),
    name STRING,
    email STRING,
    country STRING,
    expected_graduation_date DATE,
    PRIMARY KEY (country, expected_graduation_date, id))
    PARTITION BY LIST (country)(
        PARTITION australia VALUES IN ('AU','NZ') PARTITION BY RANGE (expected_graduation_date)(PARTITION graduated_au VALUES FROM (MINVALUE) TO ('2017-08-15'), PARTITION current_au VALUES FROM ('2017-08-15') TO (MAXVALUE)),
        PARTITION north_america VALUES IN ('US','CA') PARTITION BY RANGE (expected_graduation_date)(PARTITION graduated_us VALUES FROM (MINVALUE) TO ('2017-08-15'), PARTITION current_us VALUES FROM ('2017-08-15') TO (MAXVALUE))
    );
~~~

Subpartition names must be unique within a table. In our example, even though `graduated` and `current` are sub-partitions of distinct partitions, they still need to be uniquely named. Hence the names `graduated_au`, `graduated_us`, and `current_au` and `current_us`.

#### Step 5. Create and apply corresponding zone configurations

To create zone configurations and apply them to corresponding partitions, use the [`ALTER PARTITION ... CONFIGURE ZONE`]({% link {{ page.version.version }}/alter-partition.md %}#create-a-replication-zone-for-a-partition) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER PARTITION current_us OF TABLE students
    CONFIGURE ZONE USING constraints='[+ssd,+az=us1]';
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER PARTITION graduated_us OF TABLE students CONFIGURE ZONE
    USING constraints='[+hdd,+az=us1]';
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER PARTITION current_au OF TABLE students
    CONFIGURE ZONE USING constraints='[+ssd,+az=aus1]';
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER PARTITION graduated_au OF TABLE students CONFIGURE ZONE
    USING constraints='[+hdd,+az=aus1]';
~~~

#### Step 6. Verify table partitions

{% include_cached copy-clipboard.html %}
~~~ sql
> SHOW RANGES FROM TABLE students;
~~~

You should see the following output:

~~~
+-----------------+-----------------+----------+----------+--------------+
| start_key       | end_key         | range_id | replicas | lease_holder |
+-----------------+-----------------+----------+----------+--------------+
| NULL            | /"AU"           |      260 | {1,2,3}  |            1 |
| /"AU"           | /"AU"/17393     |      268 | {1,2,3}  |            1 |
| /"AU"/17393     | /"AU"/PrefixEnd |      266 | {1,2,3}  |            1 |
| /"AU"/PrefixEnd | /"CA"           |      267 | {1,2,3}  |            1 |
| /"CA"           | /"CA"/17393     |      265 | {1,2,3}  |            1 |
| /"CA"/17393     | /"CA"/PrefixEnd |      261 | {1,2,3}  |            1 |
| /"CA"/PrefixEnd | /"NZ"           |      262 | {1,2,3}  |            3 |
| /"NZ"           | /"NZ"/17393     |      284 | {1,2,3}  |            3 |
| /"NZ"/17393     | /"NZ"/PrefixEnd |      282 | {1,2,3}  |            3 |
| /"NZ"/PrefixEnd | /"US"           |      283 | {1,2,3}  |            3 |
| /"US"           | /"US"/17393     |      281 | {1,2,3}  |            3 |
| /"US"/17393     | /"US"/PrefixEnd |      263 | {1,2,3}  |            1 |
| /"US"/PrefixEnd | NULL            |      264 | {1,2,3}  |            1 |
+-----------------+-----------------+----------+----------+--------------+
(13 rows)

Time: 11.586626ms
~~~

### Repartition a table

Consider the partitioned table of students of RoachLearn. Suppose the table has been partitioned on range to store the current students on fast and expensive storage devices (example: SSD) and store the data of the graduated students on slower, cheaper storage devices(example: HDD). Now suppose we want to change the date after which the students will be considered current to `2018-08-15`. We can achieve this by using the [`PARTITION BY`]({% link {{ page.version.version }}/alter-table.md %}#partition-by) subcommand of the [`ALTER TABLE`]({% link {{ page.version.version }}/alter-table.md %}) command.

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER TABLE students_by_range PARTITION BY RANGE (expected_graduation_date) (
    PARTITION graduated VALUES FROM (MINVALUE) TO ('2018-08-15'),
    PARTITION current VALUES FROM ('2018-08-15') TO (MAXVALUE));
~~~

### Unpartition a table

You can remove the partitions on a table by using the [`PARTITION BY NOTHING`]({% link {{ page.version.version }}/alter-table.md %}#partition-by) syntax:

{% include_cached copy-clipboard.html %}
~~~ sql
> ALTER TABLE students PARTITION BY NOTHING;
~~~

### Show the replication zone for a partition

To view the replication zone for a partition, use the [`SHOW ZONE CONFIGURATION`]({% link {{ page.version.version }}/show-zone-configurations.md %}) statement.

## Locality–resilience tradeoff

There is a tradeoff between making reads/writes fast and surviving failures. Consider a partition with three replicas of `roachlearn.students` for Australian students.

- If only one replica is pinned to an Australian availability zone, then reads may be fast (via leases follow the workload) but writes will be slow.
- If two replicas are pinned to an Australian availability zone, then reads and writes will be fast (as long as the cross-ocean link has enough bandwidth that the third replica doesn’t fall behind). If those two replicas are in the same availability zone, then the loss of one availability zone can lead to data unavailability, so some deployments may want two separate Australian availability zones.
- If all three replicas are in Australian availability zones, then three Australian availability zones are needed to be resilient to an availability zone loss.

## How CockroachDB's partitioning differs from other databases

Other databases use partitioning for three additional use cases: secondary indexes, sharding, and bulk loading/deleting. CockroachDB addresses these use-cases not by using partitioning, but in the following ways:

- **Changes to secondary indexes:** CockroachDB solves these changes through online schema changes. Online schema changes are a superior feature to partitioning because they require zero-downtime and eliminate the potential for consistency problems.
- **Sharding:** CockroachDB automatically shards data as a part of its distributed database architecture.
- **Bulk Loading & Deleting:** CockroachDB does not have a feature that supports this use case as of now.
- **Logical structure of partitions:** CockroachDB uses the partitioning concept to allow users to logically subdivide a single physical table to enable independent configuration of those partitions using [zone configurations]({% link {{ page.version.version }}/configure-replication-zones.md %}). Other databases sometimes implement partitioning as the logical union of physically separate tables. This difference means that CockroachDB is able to permit inexpensive repartitioning in contrast to other databases. Consequently, CockroachDB is unable to provide the same bulk data deletion operations over table partitions that other databases achieve by physically dropping the underlying table represented by the partition.

## Known limitations

- {% include {{ page.version.version }}/known-limitations/partitioning-with-placeholders.md %}

- {% include {{ page.version.version }}/known-limitations/drop-single-partition.md %}

## See also

- [`CREATE TABLE`]({% link {{ page.version.version }}/create-table.md %})
- [`ALTER TABLE`]({% link {{ page.version.version }}/alter-table.md %})
- [Computed Columns]({% link {{ page.version.version }}/computed-columns.md %})
- [Troubleshoot Replication Zones]({% link {{ page.version.version}}/troubleshoot-replication-zones.md %})
