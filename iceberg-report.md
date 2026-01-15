
Looking into Iceberg was a step we wanted to explore because we currently don't have the raw processed Dataframe stored persistently anywhere. Currently, we only have the so-called "views" being the `.csv` files that are uploaded into S3 after the invoicing script runs but it would be useful to have a model to run queries on such as:

> "How much did this project cost in its lifetime?"

> "How many discounts did it use"

Among other queries that needs information about the project in its entirety rather than the monthly snapshots we get from the generated `.csv`

Iceberg is a good solution because it:

1. Persists data such as PARQUET files in a table format
2.  Iceberg lets you query all historical data in one place
3.  You can run SQL like SELECT SUM(cost) WHERE project_id = X across all months
4. Iceberg supports schema evolution — add/rename columns without breaking old data. Which makes migration and stuff like that still kind of annoying but less annoying.
5. Iceberg has snapshots so you can query the table as it looked at any past point.

## Iceberg Component Overview


![Iceberg Architecture](iceberg-metadata.png)


Iceberg is not something you download or some software you install in the traditional sense it really is just a format, a set of definitions that defines how data files such (Parquet, ORC, Avro) are organized, tracked, and managed, acting as a metadata layer on top of your data lake storage.

Iceberg requires 3 main components in order to function:

1. Catalog (manages table metadata)
2. Storage (storing data files in the data layer i.e. S3)
3. Query Engine (process queries)

**The tools we use for our Iceberg setup will HEAVILY depend our use cases.**

This Iceberg service will only be used for internal processes and we don't foresee that it will  be open to users. At most there will be one writer **(invoicing)** and roughly around 3 readers **(manually running queries, telemetry, coldfront)**

So with one writer and three readers we do need some concurrency guarantees. 

Though we want to find a solution with the least amount of managed services, it is equally important that these solutions are well-documented with a good ecosystem and integrate well into Iceberg. 

So for the choice of tooling I am balancing a few things:

- Minimal managed services
- Has ACID transactions
- Concurrent reads
- Integrates with Iceberg well
- Well documented with a good ecosystem & community


### Choosing a Catalog: JDBC Catalog + Postgres

Iceberg by its nature is extremely unopinionated, it was designed to fit into whatever infrastructure that already exists.

There are a whole host of solutions for the Catalog component some popular ones are the Apache Hive and the Hive Metastore, then cloud providers created their own managed solutions hence the creation of AWS Glue and Google BigLake, there are more modern and complex solutions such as Nessie and Polaris that do stuff like `git` style branching and version control etc. 

Because there were so many ways to Catalog it became a nightmare for tool developers to support them all which is why the REST Catalog came along it was meant to be a push to standardization.

Catalogs are a sum of two components: the interface and the database. Nessie, JDBC, REST catalog and the rest can be though of as the interface. The database is something that actually stores and manages the table metadata. Databases are usually some relational DB such as Postgres or MySQL

Initially for the catalog I was going to use **Hadoop Catalog**, it is the most basic and easy setup and forgoes the need to have a database because it uses the file system itself as a source of truth but it lacks strong ACID guarantees for writes because it relies on file system atomic renames. These are unreliable on object stores like S3 because they only simulate this via copy-and-delete, making it prone to data inconsistency in production. 

We don't need all the complicated setup required by Hive Metastore nor do we need the complex features that is provided by something like Nessie so the next easiest thing is to use a **JDBC Catalog**. 

JDBC is not a separate service you interface with it is essentially a **Direct Connection** to the Catalog Database, it is not a separate service you log into or anything like that. It is a piece of code that lives inside your query engine that gives the query engine direct access. It cuts out the middle man in other setups with Hive or Nessie which talk to a server, which then talks to the database.

Since we can use whatever relational database we need, I've just gone with the one I am most familiar with **Postgres**

### Choosing a Query Engine: Apache Spark

For the query engine, I’ve settled on **Apache Spark**. While there are other more feature rich query engines like Trino or Dremio, those are often built as standalone, "always-on" distributed clusters that's overkill for our current scale. They add another layer of infrastructure to manage and are generally optimized for massive, high-concurrency SQL environments. Since our primary goal is to have one reliable writer (the invoicing script) and a handful of internal readers, Spark is much more "portable." We can run it as a simple library or a single container without needing a massive persistent cluster.

Spark is essentially the gold standard for Iceberg; because the two projects grew up together, the integration is seamless and there's plenty of documentation and there's a good community behind it. It gives us exactly what we need, being the ability to process those large historical datasets and run our cost-analysis queries without the excess complications that comes with deploying a full-scale query engine like Trino. It fits perfectly into our "minimal managed services" philosophy while still giving us the flexibility to handle schema evolution and historical snapshots as we scale.

