---
title: Migrate from GeoJSON
summary: Learn how to migrate data from GeoJSON into a CockroachDB cluster.
toc: true
docs_area: migrate
---

 CockroachDB supports efficiently storing and querying [spatial data]({% link {{ page.version.version }}/export-spatial-data.md %}).

This page has instructions for migrating data from the [GeoJSON](https://wikipedia.org/wiki/GeoJSON) format into CockroachDB using [`ogr2ogr`](https://gdal.org/programs/ogr2ogr.html) and [`IMPORT`][import].

In the example below we will import a data set with [the locations of underground storage tanks in the state of Vermont (USA)](https://anrweb.vt.gov/DEC/ERT/UST.aspx?ustfacilityid=96).

## Before You Begin

To follow along with the example below, you will need the following prerequisites:

- CockroachDB [installed]({% link {{ page.version.version }}/install-cockroachdb.md %}) and [running]({% link {{ page.version.version }}/start-a-local-cluster.md %})
- [`ogr2ogr`](https://gdal.org/programs/ogr2ogr.html)
- [Python 3](https://www.python.org)

{% include {{page.version.version}}/spatial/ogr2ogr-supported-version.md %}

## Step 1. Download the GeoJSON data

First, download the storage tank GeoJSON data:

{% include_cached copy-clipboard.html %}
~~~ shell
curl -o tanks.geojson https://geodata.vermont.gov/datasets/986155613c5743239e7b1980b45bbf36_162.geojson
~~~

## Step 2. Convert the GeoJSON data to SQL

Next, convert the GeoJSON data to SQL using the following `ogr2ogr` command:

{% include_cached copy-clipboard.html %}
~~~ shell
ogr2ogr -f PGDUMP tanks.sql -lco LAUNDER=NO -lco DROP_TABLE=OFF tanks.geojson
~~~

{% include {{page.version.version}}/spatial/ogr2ogr-supported-version.md %}

## Step 3. Host the files where the cluster can access them

Each node in the CockroachDB cluster needs to have access to the files being imported.  There are several ways for the cluster to access the data; for a complete list of the types of storage [`IMPORT`][import] can pull from, see [import file locations]({% link {{ page.version.version }}/import.md %}#import-file-location).

For local testing, you can [start a local file server]({% link {{ page.version.version }}/use-a-local-file-server.md %}).  The following command will start a local file server listening on port 3000:

{% include_cached copy-clipboard.html %}
~~~ shell
python3 -m http.server 3000
~~~

## Step 4. Prepare the database

Next, create a database to hold the storage tank data:

{% include_cached copy-clipboard.html %}
~~~ shell
cockroach sql --insecure
~~~

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE DATABASE IF NOT EXISTS tanks;
USE tanks;
~~~

## Step 5. Import the SQL

Since the file is being served from a local server and is formatted as PostgreSQL-compatible SQL, we can import the data using the following [`IMPORT PGDUMP`]({% link {{ page.version.version }}/import.md %}#import-a-postgresql-database-dump) statement:

{% include_cached copy-clipboard.html %}
~~~ sql
IMPORT PGDUMP ('http://localhost:3000/tanks.sql');
~~~

~~~
        job_id       |  status   | fraction_completed | rows | index_entries | bytes
---------------------+-----------+--------------------+------+---------------+---------
  588555793549328385 | succeeded |                  1 | 2709 |          2709 | 822504
(1 row)
~~~

## See also

- [`IMPORT`][import]
- [Export Spatial Data]({% link {{ page.version.version }}/export-spatial-data.md %})
- [Spatial tutorial]({% link {{ page.version.version }}/spatial-tutorial.md %})
- [Migrate from OpenStreetMap]({% link {{ page.version.version }}/migrate-from-openstreetmap.md %})
- [Migrate from Shapefiles]({% link {{ page.version.version }}/migrate-from-shapefiles.md %})
- [Migrate from GeoPackage]({% link {{ page.version.version }}/migrate-from-geopackage.md %})
- [Spatial indexes]({% link {{ page.version.version }}/spatial-indexes.md %})
- [Migration Overview]({% link {{ page.version.version }}/migration-overview.md %})
- [Migrate from MySQL][mysql]
- [Migrate from PostgreSQL][postgres]
- [Back Up and Restore Data]({% link {{ page.version.version }}/take-full-and-incremental-backups.md %})
- [Use the Built-in SQL Client]({% link {{ page.version.version }}/cockroach-sql.md %})
- [`cockroach` Commands Overview]({% link {{ page.version.version }}/cockroach-commands.md %})
- [Using GeoServer with CockroachDB]({% link {{ page.version.version }}/geoserver.md %})

{% comment %} Reference Links {% endcomment %}

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[import]: import.html
