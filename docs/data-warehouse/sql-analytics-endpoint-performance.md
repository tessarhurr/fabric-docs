---
title: SQL analytics endpoint performance considerations
description: Learn more about performance considerations for the SQL analytics endpoint of a lakehouse in Microsoft Fabric.
author: WilliamDAssafMSFT
ms.author: wiassaf
ms.reviewer: maprycem, amasingh
ms.date: 08/01/2024
ms.service: fabric
ms.subservice: data-warehouse
ms.topic: conceptual
ms.custom:
  - build-2023
  - ignite-2023
ms.search.form: Optimization # This article's title should not change. If so, contact engineering.
---
# SQL analytics endpoint performance considerations

**Applies to:** [!INCLUDE [fabric-se](includes/applies-to-version/fabric-se.md)]

The [!INCLUDE [fabric-se](includes/fabric-se.md)] enables you to query data in the lakehouse using T-SQL language and TDS protocol. Every lakehouse has one [!INCLUDE [fabric-se](includes/fabric-se.md)]. The number of SQL analytics endpoints in a workspace matches the number of [lakehouses](../data-engineering/lakehouse-overview.md) and [mirrored databases](../database/mirrored-database/overview.md) provisioned in that one workspace.

A background process is responsible for scanning lakehouse for changes, and keeping [!INCLUDE [fabric-se](includes/fabric-se.md)] up-to-date for all the changes committed to lakehouses in a workspace. The sync process is transparently managed by Microsoft Fabric platform. When a change is detected in a lakehouse, a background process updates metadata and the [!INCLUDE [fabric-se](includes/fabric-se.md)] reflects the changes committed to lakehouse tables. Under normal operating conditions, the lag between a lakehouse and [!INCLUDE [fabric-se](includes/fabric-se.md)] is less than one minute.

## Guidance

- Automatic metadata discovery tracks changes committed to lakehouses, and is a single instance per Fabric workspace. If you are observing increased latency for changes to sync between lakehouses and [!INCLUDE [fabric-se](includes/fabric-se.md)], it could be due to large number of lakehouses in one workspace. In such a scenario, consider migrating each lakehouse to a separate workspace as this allows automatic metadata discovery to scale.
- Parquet files are immutable by design. When there's an update or a delete operation, a Delta table will add new parquet files with the changeset, increasing the number of files over time, depending on frequency of updates and deletes. If there's no maintenance scheduled, eventually, this pattern creates a read overhead and this impacts time it takes to sync changes to [!INCLUDE [fabric-se](includes/fabric-se.md)]. To address this, schedule regular [lakehouse table maintenance operations](../data-engineering/lakehouse-table-maintenance.md#execute-ad-hoc-table-maintenance-on-a-delta-table-using-lakehouse).
- In some scenarios, you might observe that changes committed to a lakehouse are not visible in the associated [!INCLUDE [fabric-se](includes/fabric-se.md)]. For example, you might have created a new table in lakehouse, but it's not listed in the [!INCLUDE [fabric-se](includes/fabric-se.md)]. Or, you might have committed a large number of rows to a table in a lakehouse but this data is not visible in [!INCLUDE [fabric-se](includes/fabric-se.md)]. We recommend initiating an on-demand metadata sync, triggered from the SQL query editor **Refresh** ribbon option. This option forces an on-demand metadata sync, rather than waiting on the background metadata sync to finish.
:::image type="content" source="media/sql-analytics-endpoint-performance/sql-analytics-endpoint-on-demand-refresh.png" alt-text="Screenshot from the Fabric portal of the on-demand refresh button in the SQL analytics endpoint toolbar.":::

## Partition size considerations

The choice of partition column for a delta table in a lakehouse also affects the time it takes to sync changes to [!INCLUDE [fabric-se](includes/fabric-se.md)]. The number and size of partitions of the partition column are important for performance:

- A column with high cardinality (mostly or entirely made of unique values) results in a large number of partitions. A large number of partitions negatively impacts performance of the metadata discovery scan for changes. If the cardinality of a column is high, choose another column for partitioning.
- The size of each partition can also affect performance. Our recommendation is to use a column that would result in a partition of at least (or close to) 1 GB. We recommend following best practices for [delta tables maintenance](../data-engineering/lakehouse-table-maintenance.md); [optimization](../data-engineering/delta-optimization-and-v-order.md). For a python script to evaluate partitions, see [Sample script for partition details](#sample-script-for-partition-details).

A large volume of small-sized parquet files increases the time it takes to sync changes between a lakehouse and its associated [!INCLUDE [fabric-se](includes/fabric-se.md)]. You might end up with large number of parquet files in a delta table for one or more reasons:

   - If you choose a partition for a delta table with high number of unique values, it's partitioned by each unique value and might be over-partitioned. Choose a partition column that doesn't have a high cardinality, and results in individual partition size of at least 1 GB.
   - Batch and streaming data ingestion rates might also result in small files depending on frequency and size of changes being written to a lakehouse. For example, there might be small volume of changes coming through to the lakehouse and this would result in small parquet files. To address this, we recommend implementing regular [lakehouse table maintenance](../data-engineering/lakehouse-table-maintenance.md).
    
### Sample script for partition details

Use the following notebook to print a report detailing size and details of partitions underpinning a delta table.

1. First, you must provide the ABSFF path for your delta table in the variable `delta_table_path`.  
    - You can get ABFSS path of a delta table from the Fabric portal **Explorer**. Right-click on table name, then select `COPY PATH` from the list of options.
1. The script outputs all partitions for the delta table.
1. The script iterates through each partition to calculate the total size and number of files.
1. The script outputs the details of partitions, files per partitions, and size per partition in GB.

The complete script can be copied from the following code block:

  ```python
  # Purpose: Print out details of partitions, files per partitions, and size per partition in GB.
    from notebookutils import mssparkutils
  
  # Define ABFSS path for your delta table. You can get ABFSS path of a delta table by simply right-clicking on table name and selecting COPY PATH from the list of options.
    delta_table_path = "abfss://<workspace id>@<onelake>.dfs.fabric.microsoft.com/<lakehouse id>/Tables/<tablename>"
  
  # List all partitions for given delta table
  partitions = mssparkutils.fs.ls(delta_table_path)
  
  # Initialize a dictionary to store partition details
  partition_details = {}

  # Iterate through each partition
  for partition in partitions:
    if partition.isDir:
        partition_name = partition.name
        partition_path = partition.path
        files = mssparkutils.fs.ls(partition_path)
        
        # Calculate the total size of the partition

        total_size = sum(file.size for file in files if not file.isDir)
        
        # Count the number of files

        file_count = sum(1 for file in files if not file.isDir)
        
        # Write partition details

        partition_details[partition_name] = {
            "size_bytes": total_size,
            "file_count": file_count
        }
        
  # Print the partition details
  for partition_name, details in partition_details.items():
    print(f"{partition_name}, Size: {details['size_bytes']:.2f} bytes, Number of files: {details['file_count']}")

  ```

## Related content

- [Better together: the lakehouse and warehouse](get-started-lakehouse-sql-analytics-endpoint.md)
- [Synapse Data Warehouse in Microsoft Fabric performance guidelines](guidelines-warehouse-performance.md)
- [Limitations of the SQL analytics endpoint](limitations.md#limitations-of-the-sql-analytics-endpoint)