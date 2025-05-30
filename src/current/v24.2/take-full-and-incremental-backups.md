---
title: Take Full and Incremental Backups
summary: Learn how to back up and restore a CockroachDB cluster.
toc: true
docs_area: manage
---

Because CockroachDB is designed with high fault tolerance, backups are primarily needed for [disaster recovery]({% link {{ page.version.version }}/disaster-recovery-planning.md %}) (i.e., if your cluster loses a majority of its nodes). Isolated issues (such as small-scale node outages) do not require any intervention. However, as an operational best practice, **we recommend taking regular backups of your data**.

There are two main types of backups:

- [Full backups](#full-backups)
- [Incremental backups](#incremental-backups)

You can use the [`BACKUP`]({% link {{ page.version.version }}/backup.md %}) statement to efficiently back up your cluster's schemas and data to popular cloud services such as AWS S3, Google Cloud Storage, or NFS, and the [`RESTORE`]({% link {{ page.version.version }}/restore.md %}) statement to efficiently restore schema and data as necessary. For more information, see [Use Cloud Storage]({% link {{ page.version.version }}/use-cloud-storage.md %}).

{% include {{ page.version.version }}/backups/backup-to-deprec.md %}

{% include {{ page.version.version }}/backups/scheduled-backups-tip.md %}

{% include {{ page.version.version }}/backups/support-products.md %}

## Backup collections

 A _backup collection_ defines a set of backups and their metadata. The collection can contain multiple full backups and their subsequent [incremental backups](#incremental-backups). The path to a backup is created using a date-based naming scheme and stored at the [collection URI]({% link {{ page.version.version }}/backup.md %}#collectionURI-param) passed with the `BACKUP` statement.

There are some specific cases where part of the collection data is stored at a different URI:

- A [locality-aware backup]({% link {{ page.version.version }}/take-and-restore-locality-aware-backups.md %}). The backup collection will be stored according to the URIs passed with the `BACKUP` statement: `BACKUP INTO LATEST IN {collectionURI}, {localityURI}, {localityURI}`. Here, the `collectionURI` represents the default locality.
- As of v22.1, it is possible to store incremental backups at a [different URI](#incremental-backups-with-explicitly-specified-destinations) to the related full backup. This means that one or multiple storage locations can hold one backup collection.

By default, full backups are stored at the root of the collection's URI in a date-based path, and incremental backups are stored in the `/incrementals` directory. The following example shows a backup collection created using these default values, where all backups reside in one storage bucket:

~~~
Collection URI:
|—— 2022
  |—— 02
    |—— 09-155340.13/
      |—— Full backup files
[...]
|—— incrementals
  |—— 2022
  |—— 02
    |—— 25-172907.21/
      |—— 20220325
        |—— 17921.23
          |—— incremental backup files
~~~

[`SHOW BACKUPS IN {collectionURI}`]({% link {{ page.version.version }}/show-backup.md %}#view-a-list-of-the-available-full-backup-subdirectories) will display a list of the full backup subdirectories at the collection's URI.

<a name="incremental-location-structure"></a> Alternately, the following directories also constitute a backup collection. There are multiple backups in two separate URIs. Each individual backup is a full backup and its related incremental backup(s). Despite using the [`incremental_location`](#incremental-backups-with-explicitly-specified-destinations) option to store the incremental backup in an alternative location, that incremental backup is still part of this backup collection as it depends on the full backup in the first cloud storage bucket:

~~~
Collection URI
|—— 2022
  |—— 02
    |—— 09-155340.13/
      |—— Full backup files
      |—— 20220210/
        |—— 155530.50/
        |—— 16-143018.72/
          |—— Full backup files
|—— incrementals
  |—— 2022
  |—— 02
    |—— 25-172907.21/
      |—— 20220325
        |—— 17921.23
          |—— incremental backup files
~~~

~~~
Explicit Incrementals URI
|—— 2022
  |—— 02
    |—— 25-172907.21/
      |—— 20220325
        |—— 17921.23
          |—— incremental_location backup files
~~~

In the examples on this page, `{collectionURI}` is a placeholder for the storage location that will contain the example backup.

{% include {{ page.version.version }}/backups/backup-storage-collision.md %}

{% include {{ page.version.version }}/backups/storage-collision-examples.md %}

## Full backups

Full backups contain an un-replicated copy of your data and can always be used to restore your cluster. These files are roughly the size of your data and require greater resources to produce than incremental backups. You can take full backups as of a given timestamp. Optionally, you can include the available [revision history]({% link {{ page.version.version }}/take-backups-with-revision-history-and-restore-from-a-point-in-time.md %}) in the backup.

In most cases, **it's recommended to take nightly full backups of your cluster**. A cluster backup allows you to do the following:

- Restore table(s) from the cluster
- Restore database(s) from the cluster
- Restore a full cluster

[Full cluster backups]({% link {{ page.version.version }}/backup.md %}#back-up-a-cluster) include [Enterprise license keys]({% link {{ page.version.version }}/licensing-faqs.md %}#set-a-license). When you [restore]({% link {{ page.version.version }}/restore.md %}) a full cluster backup that includes Enterprise license, the Enterprise license is also restored.

{% include {{ page.version.version }}/backups/file-size-setting.md %}

### Take a full backup

To perform a full cluster backup, use the [`BACKUP`]({% link {{ page.version.version }}/backup.md %}) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
BACKUP INTO '{collectionURI}';
~~~

To restore a backup, use the [`RESTORE`]({% link {{ page.version.version }}/restore.md %}) statement, specifying what you want to restore as well as the [collection's](#backup-collections) URI:

- To restore the latest backup of a table:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    RESTORE TABLE bank.customers FROM LATEST IN '{collectionURI}';
    ~~~

- To restore the latest backup of a database:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    RESTORE DATABASE bank FROM LATEST IN '{collectionURI}';
    ~~~

- To restore the latest backup of your full cluster:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    RESTORE FROM LATEST IN '{collectionURI}';
    ~~~

    {{site.data.alerts.callout_info}}
    A full cluster restore can only be run on a target cluster that has **never** had user-created databases or tables.
    {{site.data.alerts.end}}

- To restore a backup from a specific subdirectory:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    RESTORE DATABASE bank FROM {subdirectory} IN '{collectionURI}';
    ~~~

To view the available backup subdirectories, use [`SHOW BACKUPS`]({% link {{ page.version.version }}/show-backup.md %}).

## Incremental backups

If your cluster grows too large for daily [full backups](#full-backups), you can take less frequent full backups (e.g., weekly) with daily incremental backups. Incremental backups are storage efficient and faster than full backups for larger clusters.

If you are taking backups on a regular cadence, we recommend [creating a schedule]({% link {{ page.version.version }}/create-schedule-for-backup.md %}) for your backups.

### Recommendations for incremental backup frequency

Incremental backups form chains between full backups. Each incremental backup contains only the data that has changed since a base set of backups. This base set of backups must include one full backup and can include multiple incremental backups, which are typically smaller and faster to run than full backups. You can take incremental backups either as of a given timestamp or with full [revision history]({% link {{ page.version.version }}/take-backups-with-revision-history-and-restore-from-a-point-in-time.md %}). Cockroach Labs supports up to 48 incremental backups between full backups.

### Garbage collection and backups

Incremental backups with [revision history]({% link {{ page.version.version }}/take-backups-with-revision-history-and-restore-from-a-point-in-time.md %}#create-a-backup-with-revision-history) are created by finding what data has been created, deleted, or modified since the timestamp of the last backup in the chain of backups. For the first incremental backup in a chain, this timestamp corresponds to the timestamp of the base [(full) backup](#full-backups). For subsequent incremental backups, this timestamp is the timestamp of the previous incremental backup in the chain.

[Garbage collection Time to Live (GC TTL)]({% link {{ page.version.version }}/architecture/storage-layer.md %}#garbage-collection) determines the period for which CockroachDB retains revisions of a key. If the GC TTL of the [backup's target]({% link {{ page.version.version }}/backup.md %}#targets) is shorter than the frequency at which you take incremental backups with revision history, then the revisions become susceptible to garbage collection before you have backed them up. This will cause the incremental backup with revision history to fail.

We recommend configuring the garbage collection period to be at least the frequency of incremental backups and ideally with a buffer to account for slowdowns. You can configure garbage collection periods using the `ttlseconds` [replication zone setting]({% link {{ page.version.version }}/configure-replication-zones.md %}#gc-ttlseconds).

If an incremental backup is created outside of the garbage collection period, you will receive a `protected ts verification error…`. To resolve this issue, see the [Common Errors]({% link {{ page.version.version }}/common-errors.md %}#protected-ts-verification-error) page.

{% include {{ page.version.version }}/backups/pts-schedules-incremental.md %}

### Take an incremental backup

Periodically run the [`BACKUP`][backup] command to take a full backup of your cluster:

{% include_cached copy-clipboard.html %}
~~~ sql
> BACKUP INTO '{collectionURI}';
~~~

Then, create nightly incremental backups based off of the full backups you've already created. To append an incremental backup to the most recent full backup created at the [collection's](#backup-collections) URI, use the `LATEST` syntax:

{% include_cached copy-clipboard.html %}
~~~ sql
> BACKUP INTO LATEST IN '{collectionURI}';
~~~

This will add the incremental backup to the default `/incrementals` directory at the root of the backup collection's directory. With incremental backups in the `/incrementals` directory, you can apply different lifecycle/retention policies from cloud storage providers to the `/incrementals` directory as needed.

{{site.data.alerts.callout_info}}
In v21.2 and earlier, incremental backups were stored in the same directory as their full backup (i.e., `collectionURI/subdirectory`). If an incremental backup command points to a subdirectory with incremental backups created in v21.2 and earlier, v22.1 and later will write the incremental backup to the v21.2 default location. To use the v21.2 behavior on a backup that does not already contain incremental backups in the full backup subdirectory, use the `incremental_location` option, as shown in this [example](#backup-earlier-behavior).
{{site.data.alerts.end}}

If it's ever necessary, you can then use the [`RESTORE`][restore] statement to restore your cluster, database(s), and/or table(s). Restoring from incremental backups requires previous full and incremental backups.

To restore from the latest backup in the collection, stored in the default `/incrementals` collection subdirectory, run:

{% include_cached copy-clipboard.html %}
~~~ sql
RESTORE FROM LATEST IN '{collectionURI}';
~~~

To restore from a specific backup in the collection:

{% include_cached copy-clipboard.html %}
~~~ sql
RESTORE FROM '{subdirectory}' IN '{collectionURI}';
~~~

For example:

{% include_cached copy-clipboard.html %}
~~~ sql
RESTORE FROM '2023/03/23-213101.37' IN 's3://bucket/path?AUTH=implicit';
~~~

{% include {{ page.version.version }}/backups/no-incremental-restore.md %}

{{site.data.alerts.callout_info}}
`RESTORE` will re-validate [indexes]({% link {{ page.version.version }}/indexes.md %}) when [incremental backups]({% link {{ page.version.version }}/take-full-and-incremental-backups.md %}) are created from an older version (v20.2.2 and earlier or v20.1.4 and earlier), but restored by a newer version (v21.1.0+). These earlier releases may have included incomplete data for indexes that were in the process of being created.
{{site.data.alerts.end}}

### Incremental backups with explicitly specified destinations

To explicitly control where your incremental backups go, use the [`incremental_location`]({% link {{ page.version.version }}/backup.md %}#options) option. By default, incremental backups are stored in the `/incrementals` subdirectory at the root of the collection. However, there are some advanced cases where you may want to store incremental backups in a different storage location.

In the following examples, the `{collectionURI}` specifies the storage location containing the full backup. The `{explicit_incrementalsURI}` is the alternative location that you can store an incremental backup:

{% include_cached copy-clipboard.html %}
~~~ sql
BACKUP INTO LATEST IN '{collectionURI}' AS OF SYSTEM TIME '-10s' WITH incremental_location = '{explicit_incrementalsURI}';
~~~

Although the incremental backup will be in a different storage location, it is still part of the logical [backup collection](#backup-collections).

A full backup must be present in the `{collectionURI}` in order to take an incremental backup to the alternative `{explicit_incrementalsURI}`. If there isn't a full backup present in `{collectionURI}` when taking an incremental backup with `incremental_location`, the error `path does not contain a completed latest backup` will be returned.

For details on the backup directory structure when taking incremental backups with `incremental_location`, see this [incremental location directory structure](#incremental-location-structure) example.

<a name="backup-earlier-behavior"></a>To take incremental backups that are [stored in the same way as v21.2]({% link v21.2/take-full-and-incremental-backups.md %}#backup-collections) and earlier, you can use the `incremental_location` option. You can specify the same `collectionURI` with `incremental_location` and the backup will place the incremental backups in a date-based path under the full backup, rather than in the default `/incrementals` directory:

~~~ sql
BACKUP INTO LATEST IN '{collectionURI}' AS OF SYSTEM TIME '-10s' WITH incremental_location = '{collectionURI}';
~~~

When you append incrementals to this backup, they will continue to be stored in a date-based path under the full backup.

To restore an incremental backup that was taken using the [`incremental_location` option]({% link {{ page.version.version }}/restore.md %}#incr-location), you must run `RESTORE` with the full backup's location and the `incremental_location` option referencing the location passed in the original `BACKUP` statement:

~~~ sql
RESTORE TABLE movr.users FROM LATEST IN '{collectionURI}' WITH incremental_location = '{explicit_incrementalsURI}';
~~~

For details on cloud storage URLs, see [Use Cloud Storage]({% link {{ page.version.version }}/use-cloud-storage.md %}).

## Examples

{% include {{ page.version.version }}/backups/bulk-auth-options.md %}

### Scheduled backups

You can use [`CREATE SCHEDULE FOR BACKUP`]({% link {{ page.version.version }}/create-schedule-for-backup.md %}) to set a recurring schedule for full and incremental backups.

Include the `FULL BACKUP ALWAYS` clause for a schedule to take only full backups. For example, to create a schedule for taking full cluster backups:

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE SCHEDULE core_schedule_label
    FOR BACKUP INTO 's3://{BUCKET NAME}/{PATH}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}'
    RECURRING '@daily'
    FULL BACKUP ALWAYS
    WITH SCHEDULE OPTIONS first_run = 'now';
~~~
~~~
     schedule_id     |        name         | status |         first_run         | schedule |                                                                                       backup_stmt
---------------------+---------------------+--------+---------------------------+----------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  588799238330220545 | core_schedule_label | ACTIVE | 2020-09-11 00:00:00+00:00 | @daily   | BACKUP INTO 's3://{BUCKET NAME}/{PATH}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}' WITH detached
(1 row)
~~~

For more examples on how to schedule backups that take full and incremental backups, refer to [`CREATE SCHEDULE FOR BACKUP`]({% link {{ page.version.version }}/create-schedule-for-backup.md %}).

### Exclude a table's data from backups

In some situations, you may want to exclude a table's row data from a [backup]({% link {{ page.version.version }}/backup.md %}). For example, you have a table that contains high-churn data that you would like to [garbage collect]({% link {{ page.version.version }}/architecture/storage-layer.md %}#garbage-collection) more quickly than the [incremental backup](#incremental-backups) schedule for the database or cluster holding the table. You can use the `exclude_data_from_backup = true` parameter with a [`CREATE TABLE`]({% link {{ page.version.version }}/create-table.md %}#create-a-table-with-data-excluded-from-backup) or [`ALTER TABLE`]({% link {{ page.version.version }}/alter-table.md %}#exclude-a-tables-data-from-backups) statement to mark a table's row data for exclusion from a backup.

It is important to note that the backup produced contains an empty table with the data excluded. The backup process does not attempt to contact the [KV storage layer]({% link {{ page.version.version }}/architecture/storage-layer.md %}) ranges to retrieve data when `exclude_data_from_backup = true`. Thus, backups will continue to succeed even when some ranges or regions are unavailable, if and only if, the unavailable ranges are in tables excluded from the backups.

Setting this parameter prevents the cluster or database backup from delaying [GC TTL]({% link {{ page.version.version }}/configure-replication-zones.md %}#gc-ttlseconds) on the key span for this table, and it also respects the configured GC TTL. This is useful when you want to set a shorter garbage collection window for tables containing high-churn data to avoid an accumulation of unnecessary data and improves the reliability in partially unavailable clusters.

Using the `movr` database as an example:

{% include_cached copy-clipboard.html %}
~~~ sql
SHOW TABLES;
~~~
~~~
schema_name |         table_name         | type  | owner | estimated_row_count | locality
--------------+----------------------------+-------+-------+---------------------+-----------
public      | promo_codes                | table | root  |                1021 | NULL
public      | rides                      | table | root  |                 730 | NULL
public      | user_promo_codes           | table | root  |                  58 | NULL
public      | users                      | table | root  |                 211 | NULL
public      | vehicle_location_histories | table | root  |               10722 | NULL
public      | vehicles                   | table | root  |                  69 | NULL
~~~

If the `user_promo_codes` table's data does not need to be included in future backups, you can run the following to exclude the table's row data:

{% include_cached copy-clipboard.html %}
~~~ sql
ALTER TABLE movr.user_promo_codes SET (exclude_data_from_backup = true);
~~~

Then back up the `movr` database:

{% include_cached copy-clipboard.html %}
~~~ sql
BACKUP DATABASE movr INTO 's3://{BUCKET NAME}/{PATH}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}' AS OF SYSTEM TIME '-10s';
~~~

Restore the database with a [new name]({% link {{ page.version.version }}/restore.md %}#new-db-name):

{% include_cached copy-clipboard.html %}
~~~ sql
RESTORE DATABASE movr FROM LATEST IN 's3://{BUCKET NAME}/{PATH}?AWS_ACCESS_KEY_ID={KEY ID}&AWS_SECRET_ACCESS_KEY={SECRET ACCESS KEY}' WITH new_db_name = new_movr;
~~~

Move to the new database:

{% include_cached copy-clipboard.html %}
~~~ sql
USE new_movr;
~~~

You'll find that the table schema is restored:

{% include_cached copy-clipboard.html %}
~~~ sql
SHOW CREATE user_promo_codes;
~~~
~~~
table_name    |                                                create_statement
-------------------+------------------------------------------------------------------------------------------------------------------
user_promo_codes | CREATE TABLE public.user_promo_codes (
              |     city VARCHAR NOT NULL,
              |     user_id UUID NOT NULL,
              |     code VARCHAR NOT NULL,
              |     "timestamp" TIMESTAMP NULL,
              |     usage_count INT8 NULL,
              |     CONSTRAINT user_promo_codes_pkey PRIMARY KEY (city ASC, user_id ASC, code ASC),
              |     CONSTRAINT user_promo_codes_city_user_id_fkey FOREIGN KEY (city, user_id) REFERENCES public.users(city, id)
              | ) WITH (exclude_data_from_backup = true)
~~~

However, the `user_promo_codes` table has no row data:

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM user_promo_codes;
~~~
~~~
city | user_id | code | timestamp | usage_count
-----+---------+------+-----------+--------------
(0 rows)
~~~

To create a table with `exclude_data_from_backup`, see [Create a table with data excluded from backup]({% link {{ page.version.version }}/create-table.md %}#create-a-table-with-data-excluded-from-backup).

### Advanced examples

{% include {{ page.version.version }}/backups/advanced-examples-list.md %}

## See also

- [`BACKUP`][backup]
- [`RESTORE`][restore]
- [Take and Restore Encrypted Backups]({% link {{ page.version.version }}/take-and-restore-encrypted-backups.md %})
- [Take and Restore Locality-aware Backups]({% link {{ page.version.version }}/take-and-restore-locality-aware-backups.md %})
- [Take Backups with Revision History and Restore from a Point-in-time]({% link {{ page.version.version }}/take-backups-with-revision-history-and-restore-from-a-point-in-time.md %})
- [Use the Built-in SQL Client]({% link {{ page.version.version }}/cockroach-sql.md %})
- [`cockroach` Commands Overview]({% link {{ page.version.version }}/cockroach-commands.md %})
- [Expire Past Backups]({% link {{ page.version.version }}/expire-past-backups.md %})

{% comment %} Reference links {% endcomment %}

[backup]:  {% link {{ page.version.version }}/backup.md %}
[restore]: {% link {{ page.version.version }}/restore.md %}
