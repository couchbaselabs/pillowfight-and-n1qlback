# Couchbase Data Population and Query Workload Simulation

This guide explains how to set up and use Couchbase tools (`cbc-pillowfight` and `cbc-n1qlback`) to populate a Couchbase bucket with sample data, create necessary indexes, and simulate query and read/write workloads. It covers installing Couchbase to access the `/opt/couchbase/bin` tools, populating the `pillowfight` bucket, creating indexes to optimize query performance, running N1QL queries from `n1qlback_queries.txt`, and simulating a read-heavy workload with a 90/10 read/write ratio. Best practices for managing thread counts and workload distribution are also provided.

## Table of Contents
1. [Installing Couchbase and Accessing Binaries](#installing-couchbase-and-accessing-binaries)
2. [Populating the `pillowfight` Bucket](#populating-the-pillowfight-bucket)
3. [Creating Indexes for Query Performance](#creating-indexes-for-query-performance)
4. [Creating and Running N1QL Queries with `cbc-n1qlback`](#creating-and-running-n1ql-queries-with-cbc-n1qlback)
5. [Simulating a Read/Write Workload with `cbc-pillowfight`](#simulating-a-readwrite-workload-with-cbc-pillowfight)
6. [Best Practices for Workload Distribution](#best-practices-for-workload-distribution)
7. [Summary](#summary)

---

## Installing Couchbase and Accessing Binaries

To use the `cbc-pillowfight` and `cbc-n1qlback` tools located in `/opt/couchbase/bin`, you must first install Couchbase Server on your machine. These tools are included with the Couchbase Server installation.

### Installation Steps
1. **Download and Install Couchbase Server**:
   - Follow the official Couchbase installation guide for your operating system (e.g., Linux, Windows, or macOS).
   - **Official Guide**: [Couchbase Server Installation](https://docs.couchbase.com/server/current/install/install.html)
   - Choose the appropriate version (e.g., Community or Enterprise Edition) and follow the platform-specific instructions.

2. **Locate Binaries**:
   - After installation, the Couchbase tools are typically located in `/opt/couchbase/bin` on Linux systems (or equivalent paths, such as `C:\Program Files\Couchbase\Server\bin` on Windows).
   - Ensure you have execute permissions for the binaries (e.g., `chmod +x /opt/couchbase/bin/cbc*`).

3. **Stop the Couchbase Service (Optional)**:
   - If running workloads on the same machine as the Couchbase Server, you may want to stop the Couchbase service to free up resources or avoid conflicts during testing.
   - Command to stop the service (Linux):
     ```bash
     sudo systemctl stop couchbase-server
     ```
   - Verify the service is stopped:
     ```bash
     sudo systemctl status couchbase-server
     ```
   - Note: Stopping the service is optional and depends on your testing environment. If the Couchbase cluster is running on a separate machine, this step is unnecessary.

### Notes
- Ensure you have administrative privileges to install Couchbase and manage services.
- If using a remote Couchbase cluster, you only need the client tools (`cbc-pillowfight`, `cbc-n1qlback`), which can be obtained by installing the Couchbase SDK or server.

---

## Populating the `pillowfight` Bucket

The `cbc-pillowfight` tool populates the `pillowfight` bucket in a Couchbase cluster with 1 million JSON documents for testing purposes.

### Command

```bash
/opt/couchbase/bin/cbc-pillowfight \
  --json \
  --min-size 848 \
  --max-size 1240 \
  --num-items 1000000 \
  --set-pct 100 \
  -U couchbase://{hostname}:8091/pillowfight \
  -u Administrator \
  -P password \
  --timings \
  -t 6 \
  --populate-only \
  -R
```

### Purpose
This command generates 1,000,000 JSON documents in the `pillowfight` bucket, with document sizes ranging from 848 to 1,240 bytes. It simulates a balanced workload with 50% read and 50% write operations.

### CLI Settings Explained
| **Option**          | **Description**                                                                 |
|---------------------|---------------------------------------------------------------------------------|
| `--json`            | Generates documents in JSON format.                                             |
| `--min-size 848`    | Sets the minimum document size to 848 bytes.                                    |
| `--max-size 1240`   | Sets the maximum document size to 1,240 bytes.                                  |
| `--num-items 1000000` | Specifies the total number of documents to create (1 million).                 |
| `--set-pct 100`      | Configures 100% of operations as writes (sets).          |
| `-U couchbase://{hostname}:8091/pillowfight` | Specifies the Couchbase cluster URL and target bucket (`pillowfight`). |
| `-u Administrator`   | Specifies the username for authentication (`Administrator`).                    |
| `-P password`  | Specifies the password for authentication (`password`).                    |
| `--timings`         | Enables performance timing metrics for the operation.                           |
| `-t 6`              | Uses 6 threads to perform the operation, increasing concurrency.                |
| `-R`                | Enables random document generation for varied data.                             |

### Notes
- The `-t 6` option enables multithreaded execution, leveraging 6 threads for faster data population.
- The `--timings` flag provides performance metrics, useful for benchmarking the operation.

---

## Creating Indexes for Query Performance

Before running the N1QL queries with `cbc-n1qlback`, you must create indexes on the `pillowfight` bucket to optimize query performance. The following `CREATE INDEX` statements define indexes tailored to the queries in `n1qlback_queries.txt`. The `WITH {"defer_build":true}` option defers index building until the `BUILD INDEX` command is executed, reducing immediate resource usage.

### Index Creation Commands
Run the following N1QL statements in the Couchbase Query Workbench or via the `cbq` tool (located in `/opt/couchbase/bin/cbq`) to create the indexes:

```sql
CREATE INDEX idx_field1 ON `pillowfight`(Field_1) WITH {"defer_build":true};
CREATE INDEX idx_field2 ON `pillowfight`(Field_2) WITH {"defer_build":true};
CREATE INDEX idx_field2_not_null_v1 ON `pillowfight`(Field_2) WHERE Field_2 IS NOT NULL WITH {"defer_build":true};
CREATE INDEX idx_field2_max_v1 ON `pillowfight`(Field_2,LENGTH(Field_2)) WHERE Field_2 IS NOT MISSING WITH {"defer_build":true};
CREATE INDEX idx_field2_substr_v1 ON `pillowfight`(SUBSTR(Field_2, 0, 3),Field_2) WHERE Field_2 IS NOT MISSING WITH {"defer_build":true};
CREATE INDEX idx_field1_field2 ON `pillowfight`(Field_1, Field_2) WITH {"defer_build":true};
CREATE INDEX idx_field1_like ON `pillowfight`(Field_1) WHERE Field_1 LIKE '%[a-zA-Z0-9]%' WITH {"defer_build":true};
CREATE INDEX idx_doc_id ON `pillowfight`(META().id) WITH {"defer_build":true};
CREATE INDEX idx_doc_id_like_v1 ON `pillowfight`(Field_1) WHERE META().id LIKE '[0-9]{20}' WITH {"defer_build":true};
CREATE INDEX idx_field1_length_v3 ON `pillowfight`(LENGTH(Field_1),Field_1) WHERE Field_1 IS NOT MISSING WITH {"defer_build":true};
CREATE INDEX idx_field3 ON `pillowfight`(Field_3) WITH {"defer_build":true};
CREATE INDEX idx_field2_field1 ON `pillowfight`(Field_2, Field_1) WITH {"defer_build":true};
CREATE INDEX idx_multi_fields ON `pillowfight`(Field_1, Field_2, Field_3, Field_4, Field_5) WITH {"defer_build":true};
CREATE INDEX idx_field_exists ON `pillowfight`(Field_10) WHERE Field_10 IS NOT NULL WITH {"defer_build":true};
CREATE INDEX idx_field4 ON `pillowfight`(Field_4) WITH {"defer_build":true};
CREATE INDEX idx_field4_field1_length_v1 ON `pillowfight`(Field_4,LENGTH(Field_1)) WHERE Field_4 IS NOT MISSING AND Field_1 IS NOT MISSING WITH {"defer_build":true};
CREATE INDEX idx_field3_field1 ON `pillowfight`(Field_3, Field_1) WITH {"defer_build":true};
CREATE INDEX idx_field2_complex_greater_5_v1 ON `pillowfight`(LENGTH(Field_2),Field_2) WHERE LENGTH(Field_2) > 5 AND Field_2 IS NOT MISSING WITH {"defer_build":true};
CREATE INDEX idx_field2_substring ON `pillowfight`(SUBSTR(Field_2, 0, 3)) WITH {"defer_build":true};
CREATE INDEX idx_field1_field5_v1 ON `pillowfight`(Field_1, Field_5) WHERE Field_5 IS NOT MISSING AND Field_1 IS NOT MISSING WITH {"defer_build":true};
CREATE INDEX idx_field5_field6 ON `pillowfight`(Field_5, Field_6) WITH {"defer_build":true};
CREATE INDEX idx_field1_lower ON `pillowfight`(LOWER(Field_1)) WITH {"defer_build":true};
CREATE INDEX idx_field4_group ON `pillowfight`(Field_4) WITH {"defer_build":true};
CREATE INDEX idx_field1_field5 ON `pillowfight`(Field_1, Field_5) WITH {"defer_build":true};
CREATE INDEX idx_field2_complex ON `pillowfight`(Field_2) WHERE LENGTH(Field_2) > 5 WITH {"defer_build":true};
CREATE INDEX idx_field2_regex ON `pillowfight`(Field_2) WHERE REGEXP_LIKE(Field_2, '[0-9]+[a-zA-Z ]+') WITH {"defer_build":true};
CREATE INDEX idx_field1_min_length_v1 ON `pillowfight`(LENGTH(`Field_1`)) WHERE Field_1 IS NOT NULL WITH {"defer_build":true};
CREATE INDEX idx_field10_v1 ON `pillowfight`(Field_10) WHERE Field_10 IS NOT MISSING WITH {"defer_build":true};
```

### Build Deferred Indexes
After creating the indexes, build them to make them active and usable by the queries:

```sql
BUILD INDEX ON `pillowfight`((SELECT RAW name FROM system:indexes WHERE keyspace_id = 'pillowfight' AND state = 'deferred'));
```

### Purpose of Indexes
These indexes are designed to optimize the performance of the N1QL queries in `n1qlback_queries.txt` by:
- Indexing specific fields (`Field_1`, `Field_2`, etc.) used in `WHERE`, `GROUP BY`, and `ORDER BY` clauses.
- Supporting complex conditions, such as `LIKE`, `REGEXP_LIKE`, `SUBSTR`, and `LENGTH`.
- Indexing document metadata (`META().id`) for queries filtering by document IDs.
- Using `WHERE` clauses in indexes (e.g., `Field_2 IS NOT NULL`) to create partial indexes for specific query patterns.
- The `defer_build` option delays index creation until the `BUILD INDEX` command, reducing resource usage during index creation.

### Running Index Commands
1. **Using Couchbase Query Workbench**:
   - Access the Query Workbench in the Couchbase Web Console.
   - Paste and execute each `CREATE INDEX` statement individually.
   - Run the `BUILD INDEX` command to activate all deferred indexes.

2. **Using `cbq` Tool**:
   - Run the following command to start the `cbq` shell:
     ```bash
     /opt/couchbase/bin/cbq -u Administrator -p password -e http://{hostname}:8091
     ```
   - In the `cbq` shell, execute each `CREATE INDEX` statement and the `BUILD INDEX` command.

### Notes
- Ensure the `pillowfight` bucket exists before creating indexes.
- The `BUILD INDEX` command may take time depending on the data size and cluster resources.
- Verify index creation using the Query Workbench or by querying `system:indexes`.

---

## Creating and Running N1QL Queries with `cbc-n1qlback`

The `cbc-n1qlback` tool executes a set of predefined N1QL queries stored in `n1qlback_queries.txt` against the `pillowfight` bucket to simulate a static query load. The queries are separated by newlines (`\n`), not formatted as JSON (no square brackets or commas between query objects).

### Populating `n1qlback_queries.txt`
To create the `n1qlback_queries.txt` file, copy and paste the following sample queries into a text editor and save the file to a location of your choice (e.g., `/path/to/n1qlback_queries.txt`). Each query is a single line, separated by a newline (`\n`).

```
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE Field_1 LIKE '[a-zA-Z0-9]%' LIMIT 10"}
{"statement":"SELECT COUNT(*) AS count FROM `pillowfight` WHERE Field_2 IS NOT MISSING"}
{"statement":"SELECT Field_1, COUNT(*) AS count FROM `pillowfight` WHERE Field_1 IS NOT MISSING GROUP BY Field_1 ORDER BY count DESC LIMIT 5"}
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE META().id LIKE '[0-9]{20}' LIMIT 10"}
{"statement":"SELECT AVG(LENGTH(Field_1)) AS avg_length FROM `pillowfight` WHERE Field_1 IS NOT MISSING", "n1qlback":{"prepare":true}}
{"statement":"SELECT META().id, Field_3 FROM `pillowfight` WHERE Field_3 LIKE '[a-zA-Z0-9]%' LIMIT 10"}
{"statement":"SELECT META().id, Field_2 FROM `pillowfight` WHERE Field_2 IS NOT MISSING  ORDER BY Field_2 DESC LIMIT 10"}
{"statement":"SELECT COUNT(*) AS count FROM `pillowfight` WHERE Field_10 IS NOT MISSING"}
{"statement":"SELECT Field_1 FROM `pillowfight` WHERE Field_1 IS NOT MISSING GROUP BY Field_1 LIMIT 10"}
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE LENGTH(Field_1) > 10 AND Field_1 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT META().id, Field_2 FROM `pillowfight` WHERE Field_2 IN ['no2tiex9kB78sRYb', 'QR2TNYFzOAcjW 32', 'L4Dsf0vQFC0 6d9t'] LIMIT 10"}
{"statement":"SELECT Field_2, MAX(LENGTH(Field_2)) AS max_length FROM `pillowfight` WHERE Field_2 IS NOT MISSING GROUP BY Field_2 LIMIT 5"}
{"statement":"SELECT META().id, Field_1, Field_3 FROM `pillowfight` WHERE Field_1 IS NOT MISSING AND Field_3 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT LENGTH(Field_1) AS length, COUNT(*) AS count FROM `pillowfight` WHERE Field_1 IS NOT MISSING AND LENGTH(Field_1) > 0 GROUP BY LENGTH(Field_1) ORDER BY length"}
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE REGEXP_LIKE(Field_1, '[a-zA-Z0-9 ]+') LIMIT 10"}
{"statement":"SELECT META().id, Field_5 FROM `pillowfight` WHERE Field_5 IS NOT MISSING AND Field_1 IS NOT MISSING ORDER BY Field_1 LIMIT 10"}
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE LENGTH(Field_1) BETWEEN 5 AND 15 LIMIT 10"}
{"statement":"SELECT META().id, Field_4 FROM `pillowfight` WHERE Field_4 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT Field_1, COUNT(*) AS count FROM `pillowfight` WHERE Field_3 IS NOT MISSING AND Field_1 IS NOT MISSING GROUP BY Field_1 ORDER BY Field_1 LIMIT 5"}
{"statement":"SELECT META().id, Field_2 FROM `pillowfight` WHERE SUBSTR(Field_2, 0, 3) = 'no2' AND Field_2 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT MIN(LENGTH(Field_1)) AS min_length FROM `pillowfight` WHERE Field_1 IS NOT MISSING"}
{"statement":"SELECT META().id, Field_5, Field_6 FROM `pillowfight` WHERE Field_5 IS NOT MISSING AND Field_6 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE LOWER(Field_1) LIKE 'test%' AND Field_1 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT COUNT(*) AS count FROM `pillowfight` WHERE Field_1 IS NOT MISSING AND Field_2 IS NULL"}
{"statement":"SELECT Field_4, SUM(LENGTH(Field_1)) AS total_length FROM `pillowfight` WHERE Field_4 IS NOT MISSING AND Field_1 IS NOT MISSING GROUP BY Field_4 LIMIT 5", "n1qlback":{"prepare":true}}
{"statement":"SELECT META().id, Field_7 FROM `pillowfight` WHERE Field_7 LIKE '%[a-zA-Z0-9 ]%' AND Field_7 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT MAX(LENGTH(Field_2)) AS max_length FROM `pillowfight` WHERE LENGTH(Field_2) > 5  AND Field_2 IS NOT MISSING "}
{"statement":"SELECT META().id, Field_1, Field_5 FROM `pillowfight` WHERE Field_1 IS NOT MISSING AND Field_5 IS NOT MISSING ORDER BY Field_1, Field_5 LIMIT 10"}
{"statement":"SELECT COUNT(Field_3) AS unique_count FROM `pillowfight` WHERE Field_3 IS NOT MISSING GROUP BY Field_3"}
{"statement":"SELECT META().id, Field_2 FROM `pillowfight` WHERE REGEXP_LIKE(Field_2, '[0-9]+[a-zA-Z ]+') LIMIT 10", "n1qlback":{"prepare":true}}
{"statement":"SELECT Field_1, COUNT(*) AS count FROM `pillowfight` WHERE Field_1 IS NOT MISSING GROUP BY Field_1 HAVING COUNT(*) > 1 LIMIT 5"}
{"statement":"SELECT META().id, Field_2 FROM `pillowfight` WHERE LENGTH(Field_2) > 10 AND Field_2 IS NOT MISSING LIMIT 10"}
{"statement":"SELECT META().id, Field_1 FROM `pillowfight` WHERE Field_1 NOT LIKE '%[0-9][0-9][0-9]%' LIMIT 10"}
{"statement":"SELECT AVG(LENGTH(Field_4)) AS avg_length FROM `pillowfight` WHERE Field_4 IS NOT MISSING"}
{"statement":"SELECT Field_2, COUNT(*) AS count FROM `pillowfight` WHERE Field_2 IS NOT MISSING GROUP BY Field_2 ORDER BY count DESC LIMIT 5"}
```

**Instructions**:
1. Open a text editor (e.g., `nano`, `vim`, or VS Code).
2. Copy and paste the queries above into the editor.
3. Save the file as `n1qlback_queries.txt` in a directory of your choice (e.g., `/path/to/n1qlback_queries.txt`).
4. Ensure each query is on a single line, with no extra commas or square brackets, as `cbc-n1qlback` uses newlines (`\n`) to separate queries.

### Command to Run Queries

```bash
/opt/couchbase/bin/cbc-n1qlback \
  -f /path/to/n1qlback_queries.txt \
  -U couchbase://{hostname}:8091/pillowfight \
  -u Administrator \
  -P password \
  -t 4 \
  --error-log errors.log
```

### Purpose
This command executes the N1QL queries in `n1qlback_queries.txt` against the `pillowfight` bucket, simulating a static query load. It uses 4 threads for concurrent execution and logs errors to a file.

### CLI Settings Explained
| **Option**          | **Description**                                                                 |
|---------------------|---------------------------------------------------------------------------------|
| `-f /path/to/n1qlback_queries.txt` | Specifies the path to the file containing N1QL queries.                    |
| `-U couchbase://{hostname}:8091/pillowfight` | Specifies the Couchbase cluster URL and target bucket (`pillowfight`). |
| `-u Administrator`   | Specifies the username for authentication (`Administrator`).                    |
| `-P password`  | Specifies the password for authentication (`password`).                    |
| `-t 4`              | Uses 4 threads to execute queries concurrently, improving performance.          |
| `--error-log errors.log` | Logs errors encountered during query execution to `errors.log`.             |

### Query File Overview
The `n1qlback_queries.txt` file contains 35 N1QL queries that perform operations such as:
- **Filtering**: Retrieving documents based on conditions (e.g., `Field_1 LIKE '[a-zA-Z0-9]%'`).
- **Aggregation**: Computing counts, averages, minimums, and maximums (e.g., `SELECT AVG(LENGTH(Field_1))`).
- **Grouping**: Grouping by fields and applying conditions (e.g., `GROUP BY Field_1 HAVING COUNT(*) > 1`).
- **Pattern Matching**: Using `LIKE` and `REGEXP_LIKE` for string searches.
- **Sorting and Limiting**: Ordering results and limiting output (e.g., `ORDER BY Field_2 DESC LIMIT 10`).
- **Metadata Access**: Retrieving document IDs using `META().id`.

Some queries include `"n1qlback":{"prepare":true}` to precompile the query for improved performance during repeated execution.

---

## Simulating a Read/Write Workload with `cbc-pillowfight`

To simulate a read-heavy workload with a 90/10 read/write ratio, use `cbc-pillowfight` with a rate limit to stress-test the Couchbase cluster. This workload mimics a common use case where reads dominate writes.

### Command

```bash
/opt/couchbase/bin/cbc-pillowfight \
  --json \
  --min-size 848 \
  --max-size 1240 \
  --num-items 1000000 \
  --set-pct 10 \
  --rate-limit 10000 \
  -U couchbase://{hostname}:8091/pillowfight \
  -u Administrator \
  -P password \
  --timings \
  -t 6 \
  -n \
  -R
```

### Purpose
This command generates a workload with 90% read operations and 10% write operations, limited to 10,000 operations per second. It targets the `pillowfight` bucket and uses 6 threads for concurrency.

### CLI Settings Explained
| **Option**          | **Description**                                                                 |
|---------------------|---------------------------------------------------------------------------------|
| `--json`            | Generates documents in JSON format.                                             |
| `--min-size 848`    | Sets the minimum document size to 848 bytes.                                    |
| `--max-size 1240`   | Sets the maximum document size to 1,240 bytes.                                  |
| `--num-items 1000000` | Specifies the total number of documents to operate on (1 million).             |
| `--set-pct 10`      | Configures 10% of operations as writes (sets) and 90% as reads (gets).          |
| `--rate-limit 10000` | Limits the operation rate to 10,000 operations per second.                     |
| `-U couchbase://{hostname}:8091/pillowfight` | Specifies the Couchbase cluster URL and target bucket (`pillowfight`). |
| `-u Administrator`   | Specifies the username for authentication (`Administrator`).                    |
| `-P password`  | Specifies the password for authentication (`password`).                    |
| `--timings`         | Enables performance timing metrics for the operation.                           |
| `-t 6`              | Uses 6 threads to perform the operation, increasing concurrency.                |
| `-R`                | Enables random document generation for varied data.                             |

### Notes
- The `--set-pct 10` option creates a 90/10 read/write ratio, simulating a read-heavy workload common in many applications (e.g., web applications with frequent data retrievals).
- The `--rate-limit 10000` caps the operation rate at 10,000 per second. To stress the system further, increase this value and adjust the `-t` (threads) parameter to match the available CPU cores.
- For example, on an 8-core machine, set `-t 8` to fully utilize CPU resources.

---

## Best Practices for Workload Distribution

When running `cbc-pillowfight` and `cbc-n1qlback` concurrently, especially on the same machine, follow these best practices to optimize performance and avoid resource contention:

1. **Match Thread Count to CPU Cores**:
   - The `-t` option specifies the number of threads for both `cbc-pillowfight` and `cbc-n1qlback`. Set this value to match the number of available CPU cores on the machine.
   - Example: On an 8-core machine, use `-t 8` for `cbc-pillowfight`. If running `cbc-n1qlback` on the same machine, split the threads (e.g., `-t 4` for each process, totaling 8 threads).

2. **Distribute Workloads Across Machines**:
   - To avoid overloading a single machine, run `cbc-pillowfight` and `cbc-n1qlback` on separate machines. This ensures that query and read/write workloads don’t compete for CPU, memory, or network resources.
   - Example: Run `cbc-pillowfight` on Machine A and `cbc-n1qlback` on Machine B, both targeting the same Couchbase cluster.

3. **Adjust Rate Limits for Stress Testing**:
   - The `--rate-limit` option in `cbc-pillowfight` controls the operation rate. Start with a moderate value (e.g., 10,000) and increase it to stress the system. Monitor cluster performance (e.g., CPU usage, disk I/O, network latency) to identify bottlenecks.
   - When increasing the rate limit, ensure the thread count (`-t`) is sufficient to handle the increased workload.

4. **Monitor and Log Errors**:
   - Use the `--error-log` option in `cbc-n1qlback` to capture query failures for troubleshooting.
   - For `cbc-pillowfight`, the `--timings` option provides performance metrics to analyze throughput and latency.

5. **Stop Couchbase Service (if Necessary)**:
   - If running workloads on the same machine as the Couchbase Server, consider stopping the Couchbase service (`sudo systemctl stop couchbase-server`) to free up resources, especially during high-intensity tests.

---

## Summary
- **Installation**: Install Couchbase Server to access `/opt/couchbase/bin` tools. Optionally stop the Couchbase service to free resources during testing. See the [Couchbase Installation Guide](https://docs.couchbase.com/server/current/install/install.html).
- **Data Population**: Use `cbc-pillowfight` to populate the `pillowfight` bucket with 1 million JSON documents (848–1,240 bytes) using 6 threads and a 50/50 read/write ratio.
- **Index Creation**: Create and build indexes on the `pillowfight` bucket to optimize query performance before running `cbc-n1qlback`.
- **Query Workload**: Create `n1qlback_queries.txt` by copying the provided queries, then use `cbc-n1qlback` to execute 35 N1QL queries, simulating a static query load with 4 threads and error logging.
- **Read/Write Workload**: Simulate a 90/10 read/write workload with `cbc-pillowfight` using `--set-pct 10` and `--rate-limit 10000`. Adjust threads (`-t`) and rate limits to stress the system.
- **Best Practices**: Match thread counts to CPU cores, distribute workloads across machines, and monitor performance to optimize testing.

This guide provides a comprehensive, well-structured resource for setting up and running Couchbase workloads. Let me know if you need further refinements, additional examples, or assistance with specific configurations!
