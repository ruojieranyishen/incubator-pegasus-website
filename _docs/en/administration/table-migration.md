---
permalink: administration/table-migration
---

Table migration transfers all data from a table in one Pegasus cluster to another. Four methods are currently supported:

1. Shell Tool `copy_data` Migration
2. Cold Backup Migration
3. Dual Writes with Bulkload
4. Duplication Migration

# Shell Tool `copy_data` Migration

## Principle

The `copy_data` command works by scanning data from the source table and writing to the target table using set operations. This is similar to creating a client to read data from the original table and write it to a new table. Existing data in target partitions will be overwritten.

## Procedure

Use the `copy_data` command as follows:

```C++
         copy_data              <-c|--target_cluster_name str> <-a|--target_app_name str>
                                [-p|--partition num] [-b|--max_batch_count num] [-t|--timeout_ms num]
                                [-h|--hash_key_filter_type anywhere|prefix|postfix]
                                [-x|--hash_key_filter_pattern str]
                                [-s|--sort_key_filter_type anywhere|prefix|postfix|exact]
                                [-y|--sort_key_filter_pattern str]
                                [-v|--value_filter_type anywhere|prefix|postfix|exact]
                                [-z|--value_filter_pattern str] [-m|--max_multi_set_concurrency]
                                [-o|--scan_option_batch_size] [-e|--no_ttl] [-n|--no_overwrite]
                                [-i|--no_value] [-g|--geo_data] [-u|--use_multi_set]
```

Assume the original cluster is ClusterA, and the target cluster is ClusterB. The table to be migrated is TableA. Migration steps:

- Create table in target cluster: The copy_data command doesn't automatically create tables in the target cluster. Manually create the table first. The new table can have different name and partition count. Assume the new table in ClusterB is named TableB.
- Configure target cluster in Shell tool: The copy_data command requires specifying target cluster using -c parameter. Add the target cluster configuration by modifying `src/shell/config.ini` in the pegasus directory. Append these lines (replace ClusterB with actual cluster name).

```C++
[pegasus.clusters]
    ClusterB = {your_meta_server_address_list}
```

- Execute commands in Shell.

```C++
>>> use TableA
>>> copy_data -c ClusterB -a TableB -t 10000
```

- If all previous steps are configured correctly, the copy operation will start. Progress updates will be printed every second. Typically, the data copy speed should exceed 100,000+ records per second. If the process terminates due to errors (e.g., pegauss write throttling, rocksdb write stall, etc.), troubleshoot the issues before re-running the command.

# Cold Backup Migration

## Principle

Cold backup migration leverages Pegasus' [cold backup feature](/administration/cold-backup) to backup data to HDFS or other storage media, then import it to a new table via `restore` or `bulkload`.

Advantages:

- Faster speed: Cold backup copies sst files directly, significantly faster than `copy_data`.
- Higher fault tolerance: The cold backup mechanism includes robust error-handling logic to mitigate network fluctuations and other transient issues. Unlike copy_data, failures don't require restarting from scratch.
- Multi-destination friendly: A single backup can be restored to multiple targets, reducing redundant data transfers.

## Procedure

**Cold backup consists of two main phases:**

1. **Checkpoint** **creation**: All primary replicas of the table create checkpoints. Larger partitions may cause higher disk I/O pressure, potentially triggering transient read/write spikes.
2. **HDFS** **upload**: Invokes HDFS APIs to upload data after checkpoints are created. Unlimited upload speed may saturate network capacity.

**Best practices：**

- Preemptively set rate limits using the Pegasus Shell tool before cold backup to avoid network congestion.

```C++
#Configuration Method for Versions 2.3.x and Earlier
remote_command -t replica-server  nfs.max_send_rate_megabytes 50
#Configuration Method for Versions 2.4.x and Later
remote_command -t replica-server  nfs.max_send_rate_megabytes_per_disk 50
```

- Initiate a cold backup via admin-cli and wait, with parameters in order: table ID, HDFS region, HDFS storage path.

```C++
backup 3 hdfs_zjy /user/pegasus/backup
```

- The HDFS region parameter matches the following configuration in **config.ini** to connect to HDFS.

```C++
[block_service.hdfs_zjy]
type = hdfs_service
args = hdfs://zjyprc-hadoop /
```

- Monitor disk I/O metrics. When it gradually decreases, the cold backup enters phase 2. Adjust rate limits incrementally (recommended: +50 per step) based on real-time network bandwidth usage.

```C++
#Configuration Method for Versions 2.3.x and Earlier
remote_command -t replica-server  nfs.max_send_rate_megabytes 100
#Configuration Method for Versions 2.4.x and Later
remote_command -t replica-server  nfs.max_send_rate_megabytes_per_disk 100
```

- Cold backup will fail if ReplicaServer nodes restart, and cannot be canceled. Monitor `cold.backup.max.upload.file.size` metric, when it drops to zero, delete the HDFS backup directory and restart the cold backup.

**Restoring Cold Backup Data to a New Table:**

Two methods exist for restoring cold backup data to a new table:

1. Restore via restore command (described in the [Cold Backup](/administration/cold-backup) documentation)

**Best practices:**

```C++
restore -c ClusterA -a single -i 4 -t 1742888751127 -b hdfs_zjy -r /user/pegasus/backup
```

Key considerations:

- The restore command automatically creates the table. Partition count cannot be modified during restoration.
- The restore command strictly requires the original table to exist; otherwise, the operation will fail. Therefore, if the original table is unavailable, you must use Bulkload to load data into the new table.
- Apply rate limiting (same methods as cold backup) to avoid network saturation.

1. Restore via bulkload (described in the [Bulkload](/2020/02/18/bulk-load-design.html) documentation)

**Best practices:**

- Convert cold backup data to Bulkload-compatible format using Pegasus-spark's offline split operation (usage details omitted here).
- Use Pegasus-spark's Bulkload to load processed data into Pegasus.
  - The Pegasus shell CLI also supports initiating Bulkload. Assuming the offline split-processed data resides in the `/user/pegasus/split` directory.

```C++
>>> use TableB
>>> set_app_envs rocksdb.usage_scenario bulk_load
>>> start_bulk_load -a TableB -c ClusterB -p hdfs_zjy -r /user/pegasus/split
```

# Dual Writes with Bulkload

Both `copy_data` and `cold backup` migrations only handle existing data. For incremental data synchronization without stopping writes, use dual writes combined with Bulkload (supported in v2.3.x and later).

## Principle

1. Dual writes: The application writes data to both the original and target tables to synchronize incremental data.
2. Historical data migration:
   1. The service-side migrates existing data through three steps: cold backup, offline split, and Bulkload IngestBehind, ensuring synchronization of historical data.

RocksDB’s IngestBehind assigns a `global seqno` of 0 to imported SST files, placing them at the bottom of the RocksDB engine. This ensures read consistency between incremental writes (higher seqno) and historical data (seqno=0).

## Procedure

- Create the target table with `rocksdb.allow_ingest_behind=true`，If this parameter is not specified, the IngestBehind feature will be unavailable!

```C++
create TableB -p 64 -e rocksdb.allow_ingest_behind=true
```

- Enable dual writes in the application:
  - Important: Dual writes to both tables must include retry mechanisms for write failures.
- Perform cold backup and offline split to generate Bulkload-ready data.
- Initiate Bulkload with `--ingest_behind`

```C++
>>> use TableB
>>> set_app_envs rocksdb.usage_scenario bulk_load
>>> start_bulk_load -a TableB -c ClusterB -p hdfs_zjy -r /user/pegasus/split --ingest_behind
```

- Optional rate limiting: Use `max_send_rate_megabytes` to throttle network bandwidth.
- Flexible partition count: The target table can have a different partition count from the original table.

# Duplication Migration

Supported in `v2.4.x` and later versions. For details on cross-cluster synchronization, see [duplication](/administration/duplication). Duplication migration enables seamless data migration with simplified operations.

## Procedure

- Requirement: **All clients must access** **Pegasus** **via MetaProxy**. Direct connections to Metaserver IPs are prohibited. Contact the Pegasus community for client compliance checks.
- Establish Duplication: After ensuring all clients use MetaProxy, configure `duplication` to the target cluster (setup steps omitted, see  [duplication](/administration/duplication)).
- Update MetaProxy configuration: Modify the Zookeeper configuration for MetaProxy to point to the target cluster’s MetaServer IP addresses.
- Block client requests on the original table. This forces clients to reload topology from MetaProxy.

```Bash
>>> use TableB
>>> set_app_envs replica.deny_client_request reconfig*all
```

- Verify success via metrics:
  - Original table read/write QPS drops to zero.
  - Target table read/write QPS rises to match original levels.
  - `dup.disabled_non_idempotent_write_count` remains 0 in both clusters.
  - In both the original and target clusters, `recent.read.fail.count` and `recent.write.fail.count` show a brief surge followed by a drop to zero.
- Limitation: The pegasus C++ and Python clients currently do not support MetaProxy connectivity.
