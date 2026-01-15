# Apache Iceberg - Production Stack with JDBC Catalog


- **Spark Notebook** - Includes Spark SQL (query engine built-in, no separate service)
- **MinIO** - S3-compatible object storage (development only - use S3 in production)
- **PostgreSQL** - JDBC Catalog for Iceberg metadata (simple, production-ready)

## Quick Start

Start all services:

```bash
docker-compose up -d
```

## Access Points

- **Jupyter Lab**: http://localhost:8888 (no password required)
- **MinIO Console**: http://localhost:9001 (admin/password)
- **MinIO API**: http://localhost:9000
- **PostgreSQL**: localhost:5432 (user: iceberg, password: iceberg, database: iceberg)

## Directory Structure

- `datasets/` - Sample CSV datasets from Kaggle
- `notebooks/` - Jupyter notebooks (start with `demo1_jdbc_catalog.ipynb`)

## Getting Started

1. **Start services**:
   ```bash
   docker-compose up -d
   ```

2. **Open Jupyter Lab**: http://localhost:8888

3. **Run the JDBC Catalog lesson**:
   - Open `work/demo1_jdbc_catalog.ipynb`
   - Run all cells to learn JDBC Catalog

- [Apache Iceberg Documentation](https://iceberg.apache.org/)
- [JDBC Catalog Docs](https://iceberg.apache.org/docs/latest/jdbc/)
- [Spark SQL Documentation](https://spark.apache.org/docs/latest/sql-programming-guide.html)
