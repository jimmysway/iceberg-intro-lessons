# Iceberg Report - Specific Catalog Options

We didn't like the idea of JDBC catalog because that would require us to expose a Postgres or some other RDBMS on the open internet which is sub optimal. Looking further into options for Iceberg and playing around with each of them respectively.

### How an Iceberg Commit Works

1. **Write data files** (Parquet/ORC/Avro).
2. **Write manifest files** describing those data files.
3. **Write a new metadata file** (JSON) that points at the new manifests.
4. **Atomically update the table’s “current metadata pointer.”**

I said on the previous report that:

>Initially for the catalog I was going to use **Hadoop Catalog**, it is the most basic and easy setup and forgoes the need to have a database because it uses the file system itself as a source of truth but it lacks strong ACID guarantees for writes because it relies on file system atomic renames. These are unreliable on object stores like S3 because they only simulate this via copy-and-delete, making it prone to data inconsistency in production.

This statement is misleading and only partially true.

- **Hadoop Catalog requires atomic rename.** Iceberg’s own JavaDocs state this requirement clearly. ([iceberg.apache.org](https://iceberg.apache.org/javadoc/0.12.1/org/apache/iceberg/hadoop/HadoopCatalog.html?utm_source=openai))
- - **S3‑style object stores do not have atomic rename.**  
    Hadoop’s S3A connector emulates rename as **copy + delete**, which is **non‑atomic** and can be unsafe for commit protocols that assume atomic rename. ([hadoop.apache.org](https://hadoop.apache.org/docs/r3.1.0/hadoop-aws/tools/hadoop-aws/committer_architecture.html?utm_source=openai))

So being concerned that the **Hadoop Catalog + S3/B2 can break the intended commit guarantees** is valid.

What is misleading is that the previous statement may sound like Iceberg itself “lacks ACID guarantees” on object stores, this isn't the case. 

The Iceberg Reliability doc says Iceberg provides **atomic table commits** and **serializable isolation** using snapshot-based metadata updates, i.e., a change is either **fully committed** or **not visible at all**, and readers always see a **consistent snapshot**. [iceberg.apache.org](https://iceberg.apache.org/docs/latest/reliability/).

Snapshot reads guarantee **consistency of what you _do_ read**

They don't guarantee that every writer's commit survives or that the metadata is consistent.

So technically Hadoop Catalog is not fully compatible but it is something we can work with.

That being said though this option does not have strong ACID guarantees but given we have one infrequent writer and potentially 3 concurrent readers, there are some reader-side methods that we can implement in order to mitigate the potential inconsistent states such as adding a retry when it reads missing metadata or a fallback to the previous snapshot etc.

As far as strong ACID guarantees go. We have two options that I explored one is using the Iceberg REST catalog specification with something like Polaris or Gravitino or **HIVE** with the **Hive Metastore** catalog. 

**So for now I see 3 options:**
### Hive Metastore Catalog

The **Hive Metastore** option gives us a **centralized catalog** without relying on object‑store rename semantics. It stores the current metadata pointer in the metastore (typically backed by an RDBMS, I used Postgres), which makes the commit operation **atomic at the catalog layer** rather than the filesystem layer. 

In my mind and testing it is kind of a annoying thing to navigate because it is mature but finding relevant accurate documentation might be a pain because you may need to look at multiple sources i.e. the Iceberg Doc or the Spark Doc or the HIVE documentation in order to triangulate what it is that's actually going on. But there seems to be ample resources if you're willing to dig deep.

**Pros:**

- **Stronger commit guarantees** than Hadoop Catalog on object stores.
- **Well‑understood** and widely supported (Spark, Trino, Flink).
- Uses a **transactional metastore backend** to store table metadata pointers.

**Cons:**

- Requires standing up **Hive Metastore service** and a **backing database**.
- Adds operational complexity.

**Summary:**  
Hive Metastore is a **mature and stable** approach that avoids rename‑based failure modes, but it brings **infrastructure overhead**.

### Iceberg REST Catalog (Polaris, Gravitino)

The **REST catalog** is Iceberg’s newer direction in hopes of standardizing the catalog.
implements the REST catalog spec and moves the commit point into a dedicated service that can enforce **atomic commit semantics** without relying on object‑store rename behavior. This is a new development and the REST catalog spec is still in its infancy, the ecosystem and community is just not there and I had an extremely hard time figuring out what to do. In a lot of the Polaris and Gravitino documentation it warns that the software is still not considered stable enough for production environment so I wouldn't go down this route just now.

**Pros:**

- **Strong commit guarantees** on object stores.
- **No reliance on atomic rename** in S3‑compatible storage.
- **API‑based** and typically easier to integrate across engines.
- Can be hosted in a **controlled environment** without exposing an RDBMS directly.

**Cons:**

- A newer ecosystem than Hive (less operational history).
- Requires **running a catalog service** (Polaris itself).
- Some engines may require extra config or newer versions for REST support.
- Pain in the ass to setup

**Summary:**  
Not worth it

### Hadoop Catalog

The **Hadoop Catalog** is the most lightweight option — it keeps the catalog in the filesystem itself (the Iceberg table directory), so there is **no separate service** and **no database** required.

**Pros:**
- **Simplest operationally** — no metastore or REST service.
- **Zero extra infrastructure** beyond the object store.
- Works well on **HDFS/local filesystems** where atomic rename is reliable. 

**Cons:**
- Requires that **HDFS/local filesystems** where atomic rename is reliable. **S3‑style stores (including B2) do not provide atomic rename**, so the commit path is not formally safe.
- **Concurrent writers** can lead to lost updates; even with one writer there can be **transient metadata visibility issues**.

**Summary:**  
Hadoop Catalog is the **lowest‑overhead** option, but on object stores it is **not fully compatible with strong commit semantics**.  

Given we only have **one infrequent writer**, we can likely make this work with **reader‑side retries/fallbacks**. This is likely the best option unless we like to suffer.

