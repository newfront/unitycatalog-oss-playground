# Run Unity Catalog Locally with Spark and Delta Lake

**unitycatalog-playground** is a complete, interactive environment for running
open source [Unity Catalog](https://github.com/unitycatalog/unitycatalog)
locally on macOS or Linux. It connects Unity Catalog to Apache Spark and Delta
Lake, stores catalog metadata in PostgreSQL, stores managed-table data in
S3-compatible object storage, and provides ready-to-run
[marimo](https://marimo.io/) notebooks.

Use it to create, write, and query catalog-managed Delta tables—not just start a
catalog server.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

## What you get

- **Unity Catalog OSS** for catalogs, schemas, tables, volumes, and credentials
- **Apache Spark + Delta Lake** as the local compute and table engine
- **PostgreSQL** for persistent Unity Catalog metadata
- **RustFS** for local S3-compatible managed-table storage and credential vending
- **marimo notebooks** with runnable Unity Catalog and Delta Lake examples
- **Local and remote modes** for using the bundled catalog or another UC server

This project complements the
[official Unity Catalog quickstart](https://github.com/unitycatalog/unitycatalog/blob/main/docs/quickstart.md).
The upstream quickstart is the best place to learn the server and APIs. This
playground adds the compute engine, durable metadata database, object storage,
and notebooks needed for an end-to-end local lakehouse.

## Quickstart: run Unity Catalog locally on a Mac

The container images used by the playground support both Apple Silicon
(`arm64`) and Intel (`amd64`) Macs.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with at
  least 4 GB of memory available to Spark
- [Git](https://git-scm.com/)
- [`just`](https://github.com/casey/just), the project command runner

Install and start the prerequisites with Homebrew:

```bash
brew install --cask docker
brew install just
open -a Docker
```

After Docker Desktop is running:

```bash
git clone https://github.com/open-lakehouse/unitycatalog-playground.git
cd unitycatalog-playground
just uc=local start
```

The first start builds the notebook image and may take several minutes. When the
environment is ready, the command prints a URL like:

```text
http://localhost:2718/?access_token=...
```

Open that URL, select `unitycatalog-delta.py`, and run the notebook cells in
order. You will create a Unity Catalog schema and a catalog-managed Delta table,
write sample data through Spark, and query it again.

The local services are available at:

- marimo notebook environment: <http://localhost:2718>
- Unity Catalog API: <http://localhost:8080>
- RustFS object browser: <http://localhost:9001>
- RustFS S3 API: <http://localhost:9000>

To stop the stack while preserving its metadata and table data:

```bash
just uc=local down
```

## Local and remote Unity Catalog modes

One `docker-compose.yaml` supports two modes:

- **Local:** `just uc=local ...` starts the complete stack: Unity Catalog,
  PostgreSQL, RustFS, and marimo with Spark and Delta Lake.
- **Remote:** `just ...` starts only the notebook environment. Configure
  `UC_SERVER_URL`, `UC_SERVER_PORT`, and `UC_TOKEN` in `.env` to connect it to an
  external Unity Catalog server.

Use the same mode when starting and stopping the environment:

```bash
just uc=local start  # build and start the complete local stack
just uc=local logs   # follow logs
just uc=local ps     # show service status
just uc=local down   # stop containers but preserve data

just start           # connect the notebooks to a remote UC server
just down
```

## Configuration

Run `just init` to create `.env` from [`.env.example`](.env.example). The
defaults work for the local quickstart. Edit `.env` to change Spark resources,
ports, PostgreSQL or RustFS settings, the remote UC endpoint, or proxy settings.

For environments that require package proxies:

```dotenv
PYPI_PROXY_URL=https://pypi-proxy.yourcompany.com/simple
MAVEN_PROXY_URL=https://maven-proxy.yourcompany.com/maven2
```

Docker Compose loads `.env` automatically. Leaving those values blank uses
public PyPI and Maven Central.

### Network bridge

`marimo-spark` joins an **external** docker network, `uc-shared`, so it can also
reach a UC server running in a *separate* compose project. Because the network is
declared `external: true`, it must exist before Compose starts — otherwise you'll
see `network uc-shared declared as external, but could not be found`.

`just up` / `just up-detached` / `just jars` run `just net` first, which creates
the network idempotently. You can also run it on its own:

```bash
just net   # docker network create uc-shared (no-op if it already exists)
```

## Pre-download Spark jars (optional, faster startup)

By default each notebook resolves its jars (`delta-spark`, `unitycatalog-spark`,
`hadoop-aws`, and their transitive dependencies) from Maven the **first time** a
Spark session is created. That resolution can take several minutes and needs
network access — and it repeats on a fresh container or whenever the Ivy cache
is cleared.

`just jars` does that resolution **once, up front**, and drops the jars into
`./spark/jars` (bind-mounted to `/spark/jars` in the container). A notebook that
reads from that directory then starts Spark straight off the local classpath —
no per-session Maven round-trip.

```bash
just build   # the image must exist first
just jars    # resolve + download into ./spark/jars
just up      # start the environment
```

> **Note:** `just jars` resolves through `MAVEN_PROXY_URL` (from your `.env`)
> when it is set. It uses a proxy-only Ivy resolver
> (`spark/ivysettings.xml`), avoiding attempts to reach the firewalled
> `repo1.maven.org` and Spark Packages endpoints first. When
> `MAVEN_PROXY_URL` is blank, it resolves from public Maven Central.

The downloaded jars are git-ignored (the directory is kept via
`spark/jars/.gitkeep`), so they never get committed. Re-running `just jars` is
safe—existing jars are not re-downloaded. The coordinates it fetches live in
the `jars_packages` variable at the top of the [Justfile](Justfile); keep them
in sync with the `spark.jars.packages` used in the notebooks.

### When to use it

- Use it for fast, repeatable notebook startup across container rebuilds.
- Use it behind a slow or restricted corporate network.
- Skip it for a quick one-off run; the notebooks resolve dependencies from
  Maven automatically when `./spark/jars` is empty.

## Included notebooks

- [`unitycatalog-delta.py`](marimo-playground/notebooks/unitycatalog-delta.py)
  creates a schema and a catalog-managed Delta table, generates sample data,
  writes it through Spark, and queries it.
- [`metric-views.py`](marimo-playground/notebooks/metric-views.py) demonstrates
  Unity Catalog metric views with reusable dimensions and measures.
- [`delta-new-in-4.4.0.py`](marimo-playground/notebooks/delta-new-in-4.4.0.py)
  explores Delta Lake 4.4 features on Apache Spark 4.2.
- [`external-access-unitycatalog-delta-managed-read.py`](marimo-playground/notebooks/databricks/external-access-unitycatalog-delta-managed-read.py)
  demonstrates reading a remote Databricks Unity Catalog table through external
  data access and credential vending.

Open the marimo URL printed by `just ... start`, select a notebook, and run its
cells in order. Markdown cells provide context and do not need to be executed.

## How the local stack works

Spark runs inside the marimo container and uses the Unity Catalog Spark
connector. Unity Catalog stores object definitions in PostgreSQL and vends
short-lived credentials for managed-table data in RustFS. This gives local
development the same basic separation of catalog metadata, compute, and object
storage used by a cloud lakehouse.

## Local Unity Catalog + Postgres + RustFS

`just uc=local …` starts Postgres 16.3 alongside the UC server (same defaults as
upstream [postgres-example.yml](https://github.com/unitycatalog/unitycatalog/blob/main/etc/db/postgres-example.yml)).
Hibernate is pointed at it via [`etc/conf/hibernate.properties`](etc/conf/hibernate.properties).
Catalog metadata lives in the Docker volume `uc_postgres_data` and **survives**
`just down` / restarts. Wipe it with `just uc=local clean` or
`just uc=local down-volumes`.

### RustFS S3 storage

Managed table **data** lives in [RustFS](https://rustfs.com), an S3-compatible
object store, rather than a local file path — so the local stack behaves like a
cloud UC on real object storage (`storage-root.tables=s3://uc-warehouse`). A
one-shot `rustfs-init` service creates the bucket, mints a short-lived STS
credential from RustFS, and renders it into the UC server config so UC can vend
it to Spark. (This UC build has no custom-S3-endpoint support yet, so the STS
token is minted from RustFS rather than assumed by UC directly — see
[`etc/rustfs/bootstrap.sh`](etc/rustfs/bootstrap.sh).)

```bash
just uc=local start
just uc=local rustfs-url
```

Open <http://localhost:9001> and log in with `RUSTFS_ACCESS_KEY` and
`RUSTFS_SECRET_KEY` (both default to `rustfsadmin`) to browse the objects your
notebooks write.

The vended STS credential **expires** (default 12h). If managed-table reads or
writes start failing with credential errors, refresh it — the bucket and its
data are left untouched:

```bash
just uc=local rotate-creds  # re-mint the STS credential + restart UC
```

Object data lives in the `rustfs_data` volume (wiped by
`just uc=local clean` or `just uc=local down-volumes`). Tune the bucket,
credentials, region, and STS lifetime with `RUSTFS_*`, `UC_STORAGE_BUCKET`,
`S3_REGION`, and `STS_DURATION_SECONDS` in `.env`.

## Stop or reset the environment

Stop containers and networks while keeping PostgreSQL metadata and RustFS table
data:

```bash
just uc=local down
# or: docker compose --profile local-uc down
```

To reset everything, delete the named volumes and the locally built marimo
image:

```bash
just uc=local clean
```

> [!CAUTION]
> `clean` permanently deletes the local Unity Catalog metadata and all managed
> table data stored by the playground.

## Troubleshooting

### Docker is not running

Start Docker Desktop and wait for its engine to become ready:

```bash
open -a Docker
docker info
```

### The `uc-shared` network is missing

The `just` commands create this network automatically. If Compose was invoked
directly or startup was interrupted, recreate it with:

```bash
just net
```

### The first Spark session is slow

Spark downloads Delta Lake, the Unity Catalog connector, Hadoop AWS, and their
transitive dependencies the first time a notebook initializes. Pre-download
them for subsequent runs:

```bash
just build
just jars
```

### Spark exits or runs out of memory

Give Docker Desktop at least 4 GB of memory for Spark. You can also lower or
raise `SPARK_DRIVER_MEMORY` and `SPARK_DRIVER_CORES` in `.env`, then restart the
marimo container and notebook kernel.

### Managed-table operations fail after several hours

The local RustFS STS credential expires after 12 hours by default. Refresh it
without deleting the bucket or its data:

```bash
just uc=local rotate-creds
```

## Catalog Managed Tables — Python Helpers

The helper module at
[`delta/python/catalog-managed.py`](delta/python/catalog-managed.py) provides
reusable utilities for creating and populating catalog-managed Delta tables
with PySpark. The
[`unitycatalog-delta.py`](marimo-playground/notebooks/unitycatalog-delta.py)
notebook uses the same helpers interactively.

### `Pets` dataclass

A simple dataclass representing a rescue animal record.

```python
@dataclass
class Pets:
    uuid: str    # unique identifier
    name: str    # pet name
    age: int     # age in years
    adopted: bool
```

---

### `generate_pets(batch_size=10, total=100)`

Generates `total` random `Pets` records and yields them in batches of `batch_size`.

```python
pets = generate_pets(batch_size=10, total=100)

first_batch = next(pets)        # list[Pets] with 10 records
for batch in pets:              # iterate remaining batches
    print(batch)
```

---

### `pets_to_dataframe(pets, spark)`

Converts a `list[Pets]` batch into a Spark `DataFrame` with an explicit schema.

```python
df = pets_to_dataframe(first_batch, spark)
df.show()
```

Schema:

| column    | type    | nullable |
| --------- | ------- | -------- |
| `uuid`    | string  | no       |
| `name`    | string  | no       |
| `age`     | integer | no       |
| `adopted` | boolean | no       |

---

### `create_table_ddl(table_name, schema, properties)`

Builds a `CREATE TABLE IF NOT EXISTS ... USING DELTA` DDL string from a
`StructType` schema and an optional `dict` of `TBLPROPERTIES`.

```python
props = {"delta.feature.catalogManaged": "supported"}
ddl = create_table_ddl("sanctuary.pets", df.schema, props)
print(ddl)
```

---

### `create_table_using_sql(table_name, schema, properties, spark)`

Executes the DDL produced by `create_table_ddl` via `spark.sql`. Returns an
empty `DataFrame` on success.

```python
create_table_using_sql("sanctuary.pets", df.schema, props, spark)
```

---

### End-to-end example

The following mirrors the flow in the
[`unitycatalog-delta.py`](marimo-playground/notebooks/unitycatalog-delta.py)
notebook:

```python
# 1. Create the schema
spark.sql("CREATE SCHEMA IF NOT EXISTS unity.sanctuary")

# 2. Generate pets
pets = generate_pets(batch_size=10, total=100)
first_batch = next(pets)

# 3. Derive the schema from the first batch
df = pets_to_dataframe(first_batch, spark)
props = {"delta.feature.catalogManaged": "supported"}

# 4. Create the table
create_table_using_sql("sanctuary.pets", df.schema, props, spark)

# 5. Write the first batch
df.write.format("delta").mode("append").saveAsTable("sanctuary.pets")

# 6. Write remaining batches
for batch in pets:
    pets_to_dataframe(batch, spark).write.format("delta").mode("append").saveAsTable("sanctuary.pets")

# 7. Verify
spark.sql("SELECT COUNT(*) AS total FROM sanctuary.pets").show()
```

## Frequently asked questions

### Can I run Unity Catalog locally on an Apple Silicon Mac?

Yes. The stack's Unity Catalog, Spark, PostgreSQL, RustFS, and AWS CLI container
images publish `linux/arm64` variants. Docker Desktop selects them automatically
on Apple Silicon. The same stack also supports Intel (`amd64`) Macs.

### Do I need a Databricks account?

No. Local mode uses the open source Unity Catalog server and runs entirely in
Docker. A Databricks account is needed only for examples that explicitly
connect to a Databricks-managed Unity Catalog.

### How is this different from the official Unity Catalog Docker setup?

The official repository provides the authoritative Unity Catalog server,
clients, APIs, and UI. This community playground builds on that server and adds
Spark, Delta Lake, PostgreSQL, S3-compatible storage, credential vending, and
interactive notebooks so you can exercise complete table workflows locally.

### Can the notebooks connect to another Unity Catalog server?

Yes. Run in remote mode and set `UC_SERVER_URL`, `UC_SERVER_PORT`, and
`UC_TOKEN` in `.env`:

```bash
just start
```

### Is this intended for production?

No. The playground is designed for local development, demonstrations, and
experimentation. Use production-grade identity, networking, storage,
observability, backups, and secret management for a deployed environment.

## Contributing

Issues and pull requests are welcome. When reporting a startup problem, include
your Mac architecture, Docker Desktop version, the command you ran, and the
relevant output from:

```bash
just uc=local ps
just uc=local logs
```

## License

Licensed under the [Apache License 2.0](LICENSE).
