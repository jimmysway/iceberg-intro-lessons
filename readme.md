# Apache Iceberg - Catalog Options and Notebooks


- **Spark Notebook** - Includes Spark SQL (query engine built-in, no separate service)
- **MinIO** - S3-compatible object storage (development only - use S3 in production)
- **PostgreSQL** - JDBC Catalog for Iceberg metadata (simple, production-ready)

## Quick Start

Choose one catalog stack and start it from `iceberg-versions/`.

### JDBC Catalog (PostgreSQL + MinIO)

```bash
cd iceberg-versions
docker compose --profile jdbc up -d
```

### Hadoop Catalog (S3)

```bash
cd iceberg-versions
docker compose -f docker-compose-hadoop.yml --profile hadoop up -d
```

### Hive Metastore (HMS) Catalog (PostgreSQL + MinIO + HMS)

```bash
cd iceberg-versions
docker compose -f docker-compose-hms.yml --profile hms up -d
```

## Access Points

- **JDBC stack**:
  - **Jupyter Lab**: http://localhost:8888 (no password required)
  - **MinIO Console**: http://localhost:9001 (admin/password)
  - **MinIO API**: http://localhost:9000
- **Hadoop stack**:
  - **Jupyter Lab**: http://localhost:8890 (no password required)
- **HMS stack**:
  - **Jupyter Lab**: http://localhost:8889 (no password required)
  - **MinIO Console**: http://localhost:9401 (admin/password)
  - **MinIO API**: http://localhost:9400
  - **Hive Metastore Thrift**: thrift://localhost:9083

## Catalogs Covered

This project includes notebooks for three Iceberg catalog types:

- **JDBC Catalog** - Metadata stored in PostgreSQL.
- **Hadoop Catalog** - Metadata stored in an S3 bucket.
- **HMS Catalog** - Metadata stored in a Hive Metastore service backed by PostgreSQL.

## Notes

- Ensure the MinIO `warehouse` bucket exists before running the JDBC or HMS
  catalog notebooks.
- The Hadoop catalog is connected to S3 and requires an `iceberg-versions/.env`
  file with credentials and endpoint configuration. The Hadoop notebook reads
  this file to connect to S3.

## Directory Structure

- `datasets/` - Sample CSV datasets from Kaggle
- `iceberg-versions/` - Docker Compose files for each catalog stack
- `notebooks-jdbc/` - JDBC catalog notebooks
- `notebooks-hadoop/` - Hadoop catalog notebooks (S3-backed)
- `notebooks-hms/` - HMS catalog notebooks

## Getting Started

1. **Start the stack you want** (see Quick Start).

2. **Open Jupyter Lab** for that stack (see Access Points).

3. **Run the catalog notebooks**:
   - JDBC: `notebooks-jdbc/demo1_jdbc_catalog.ipynb`
   - Hadoop: `notebooks-hadoop/demo1_hadoop_catalog.ipynb`
   - HMS: `notebooks-hms/demo1_hms_catalog.ipynb`

- [Apache Iceberg Documentation](https://iceberg.apache.org/)
- [JDBC Catalog Docs](https://iceberg.apache.org/docs/latest/jdbc/)
- [Spark SQL Documentation](https://spark.apache.org/docs/latest/sql-programming-guide.html)
