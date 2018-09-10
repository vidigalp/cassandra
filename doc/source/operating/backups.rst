.. Licensed to the Apache Software Foundation (ASF) under one
.. or more contributor license agreements.  See the NOTICE file
.. distributed with this work for additional information
.. regarding copyright ownership.  The ASF licenses this file
.. to you under the Apache License, Version 2.0 (the
.. "License"); you may not use this file except in compliance
.. with the License.  You may obtain a copy of the License at
..
..     http://www.apache.org/licenses/LICENSE-2.0
..
.. Unless required by applicable law or agreed to in writing, software
.. distributed under the License is distributed on an "AS IS" BASIS,
.. WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
.. See the License for the specific language governing permissions and
.. limitations under the License.

.. highlight:: none

Backups
=======

Cassandra is a distributed, decentralized, fault-tolerant system. Data is replicated throughout multiple nodes (centers) across various data centers. The fact that Cassandra is decentralized means that it can survive single or even multi-node failures without losing any data. With Cassandra there is no single point of failure, making Cassandra a highly available database. 

As long as there is one node containing the data, Cassandra can recover the data without resorting to an external backup. If set up right, Cassandra will be able to handle disk or other hardware failures even in case of an entire data center going down.

However, Cassandra backups are still necessary to recover from the following scenarios:

1. Errors made in data updates by client applications

2. Accidental deletions

3. A catastrophic failure that will require you to rebuild the entire cluster

4. Data corruption

5. A desire to rollback cluster to a previous known good state

Setting up a Backup Strategy
============================
When setting up your backup strategy, you should have some points into consideration.

**Secondary Storage footprint:** Backup footprint can be much larger than the live database setup depending on the frequency of backups and retention period. It is therefore vital to create an efficient storage solution that decreases storage CAPEX (capital expenditure) as much as possible.

**Recovery point objective (RPO):**  The maximum targeted period in which data might be lost from service due to a significant incident.

**Recovery time objective (RTO):** The targeted duration of time and a Service Level Agreement within which a backup must be restored after a disaster/disruption to avoid unacceptable consequences associated with a break in business continuity.

**Backup Performance:** The backup performance should be sufficient enough to at least match the data change rate in the Cassandra Cluster. 

Backup Alternatives
===================

1. Snapshot-based backups
"""""""""""""""""""""""""
The purpose of a snapshot is to make a copy of all or part of keyspaces and tables in a node and to save it into a separate file. When you take a snapshot, Cassandra first performs a flush, to push any data residing in the memtables into the disk (SStables), and then makes a hard link for each SSTable file.

Each snapshot contains a manifest.json file that lists the SSTable files included in the snapshot, to make sure that the entire contents of the snapshot are present.

Nodetool snapshot operates at the node level, meaning that you will need to run it at the same time on multiple nodes.

2. Incremental Backups
""""""""""""""""""""""
When incremental backups are enabled, Cassandra creates backups as part of the process of flushing SSTables to disk. The backup consists of a hard link to each data file that is stored in a backup directory. In Cassandra, incremental backups contain only new SStables files, making them dependent on the last snapshot created. Files created due to compaction are not hard linked.

3. Incremental Backups in Combination with Snapshots
""""""""""""""""""""""""""""""""""""""""""""""""""""
By combining both methods, you can achieve a better granularity of the backups. Data is backed up periodically via the snapshot, and incremental backup files are used to obtain granularity between scheduled snapshots.

4. Commit log Backup in Combination with Snapshot
"""""""""""""""""""""""""""""""""""""""""""""""""
This approach is a similar method to incremental backup with snapshots. Rather than relying on incremental backups to backup newly added SStables, commit logs are archived. As with the previous solution, snapshots provide the bulk of backup data, while the archive of commit log is used for point-in-time backup.

5. CommitLog Backup in Combination with Snapshot and Incremental
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
In addition to incremental backups, commit logs are archived. This process relies on a feature called Commitlog Archiving.  Like with the previous solution, Snapshots, provide the bulk of backup data, incremental complement and the archive of commit log is used for point-in-time backup.

Due to the nature of commit logs, it is not possible to restore commit logs to a different node than the one it was backed up from. This limitation restricts the scope of restoring commit logs in case of hardware catastrophic failure. (And a node is not fully restored, only its data.)

6. Datacenter Backup
""""""""""""""""""""
With this setup, Cassandra will stream data to the backup as it is added. This mechanism prevents cumbersome snapshot-based backups requiring files stored on a network. However, this will not protect from a developer mistake (e.g., deletion of data), unless there is a time buffer between both data centers.



+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Type**                         | **Advantages**                                                                                                                                                  | **Disadvantages**                                                                                                                                                                                                                                     |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Snapshot-based Backups**       | **- Simple to manage:** Requires simple scheduled snapshot command to run on each of the nodes.                                                                 | **- Potential large RPO:** Snapshots require flushing all in-memory data to disk; therefore, frequent snapshot calls will impact the cluster’s performance.                                                                                           |
|                                  |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
|                                  |                                                                                                                                                                 | **- Large storage footprint:** Depending on various factors -- such as workload type, compaction strategy, or versioning interval -- compaction may cause multifold more data to be backed up, causing an increase in Capital Expenditure (CapEx).    |
|                                  | Cassandra nodetool utility provides the clearsnapshot command that removes the snapshot files. (Auto snapshots on table drops are not visible to this command.) |                                                                                                                                                                                                                                                       |
|                                  |                                                                                                                                                                 | **- Snapshot storage management overhead:** Cassandra admins are expected to remove the snapshot files to a safe location such as AWS S3.                                                                                                             |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Incremental Backups**          | **- Better storage utilization:** There are no duplicate records in backup, as compacted files are not backed up.                                               | **- Space management overhead:** The incremental backup folders must be emptied after being backed up. Failure to do so may cause severe space issues on the cluster.                                                                                 |
|                                  |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
|                                  | **- Point-in-time backup:** Companies can achieve better RPO, as backing up from the incremental backup folder is a continuous process.                         | **- Spread across many files:** Since incremental backups create files every time a flush occurs, it typically produces many small files, making file management and recovery, not an easy task that can have an impact on RTO and the Service Level. |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Incremental Backups in**       | **- Large backup files:** Only data between snapshots are from the incremental backups.                                                                         | **- Space management overhead:** Every time a snapshot is backed up, data needs to be cleaned up.                                                                                                                                                     |
| **Combination with Snapshots**   |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
|                                  | **- Point-in-time:** It provides point-in-time backup and restores.                                                                                             | **- Operationally burdensome:** Requires DBAs to script solutions.                                                                                                                                                                                    |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **CommitLog Backup **            | **- Point in time:** It provides the best point in time backup and restores.                                                                                    | **- Space management overhead:** Every time a snapshot backed-up data needs to be cleaned up. Increased Operational Expenditure (OpEx.)                                                                                                               |
| **in Combination**               |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
| **with Snapshot**                |                                                                                                                                                                 | **- Restore Complexity:** Restore is more complicated as part of the restore will happen from the commit log replay.                                                                                                                                  |
| **and Incremental**              |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
|                                  |                                                                                                                                                                 | **- Storage overhead:** Snapshot-based backup will provide storage overhead because of duplication of data due to compaction, resulting in higher CapEx expenditure.                                                                                  |
|                                  |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
|                                  |                                                                                                                                                                 | **- Highly complex:** Due to the nature of dealing with three times of backups, plus the streaming and managing of the commit log. It is a highly sophisticated backup solution.                                                                      |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| **Datacenter Backup**            | **- Hot Backup:** It can provide a swift way to restore data.                                                                                                   | **- Additional Datacenter:** Since it requires a new datacenter to be built, it needs higher CapEx expenditure as well as OpEx expenditure.                                                                                                           |
|                                  |                                                                                                                                                                 |                                                                                                                                                                                                                                                       |
|                                  | **- Space management:** Using RF = 1, you can avoid data replication.-                                                                                          | **- Prone to Developer Mistakes:**  Will not protect from developer mistakes (unless there is a time buffer, as mentioned above).                                                                                                                     |
+----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+