

## 🔹 **Basic to Intermediate MySQL Questions**

### 1. **What is the difference between `CHAR` and `VARCHAR`?**

* `CHAR`: Fixed-length (faster for small strings).
* `VARCHAR`: Variable-length (space-efficient for long/irregular strings).

---

### 2. **How do you perform a backup of a MySQL database?**

* Using `mysqldump`:

  ```bash
mysqldump -h <HOST_OR_ENDPOINT> -u <USERNAME> -p <DATABASE_NAME> > backup.sql
  ```

```
*  -u root → username is root
*  -p → prompts you for the password (it’s not the DB name)
*   my_database → this is the name of the database you want to dump
*   > → redirects the output into a file
*   my_database.sql → the dump file that gets created


```

---

### 3. **How to list all databases and all tables in MySQL?**

```sql
SHOW DATABASES;
USE db_name;
SHOW TABLES;
```

---

### 4. **What is the difference between `INNER JOIN`, `LEFT JOIN`, and `RIGHT JOIN`?**

* `INNER JOIN`: Only matching rows
* `LEFT JOIN`: All left rows + matching right
* `RIGHT JOIN`: All right rows + matching left

---

## 🔹 **Performance & Optimization Questions**

### 5. **What is indexing in MySQL and why is it important?**

* Indexes improve **query speed** by avoiding full table scans.
* But too many indexes slow down `INSERT`/`UPDATE`.

---

### 6. **How do you check slow queries in MySQL?**

* Enable slow query log in `my.cnf`:

  ```ini
  slow_query_log = 1
  slow_query_log_file = /var/log/mysql/slow.log
  long_query_time = 2
  ```

---

### 7. **What is the difference between `MyISAM` and `InnoDB`?**

| Feature        | MyISAM  | InnoDB     |
| -------------- | ------- | ---------- |
| Transactions   | ❌ No    | ✅ Yes      |
| Foreign Keys   | ❌ No    | ✅ Yes      |
| Crash Recovery | ❌ Poor  | ✅ Strong   |
| Performance    | ✅ Reads | ✅ Balanced |

---

## 🔹 **Scenario-Based Questions**

### 8. **How do you recover from a deleted table in production?**
* First, I check if we have recent backups — like daily mysqldump, logical backups, or physical snapshots.
If yes, I restore just the deleted table using:
```
mysql -u root -p my_database < table_backup.sql
```
* If no backup exists, and the server hasn’t been restarted or compacted, I try tools like undrop-for-innodb or inspect the binary logs to recover the table data.
* In one incident, a dev accidentally dropped a reporting table — I used binlogs to replay the table creation and insert queries up to just before the drop. Since binlogs were enabled, we avoided data loss.

```

🔧 Scenario: Developer accidentally dropped a table (e.g., reporting_data)
🛠️ Goal: Use binary logs to recover it up to the point before DROP TABLE
----------------------------------------------------------------------------

* ✅ Pre-requirements:
 * Binary logging must be enabled (log_bin in my.cnf)
 * You know approximate time when the table was dropped

* 📌 Recovery Steps:
--------------------
1. ✅ Find the right binlog file and time window
---------------------------------------------------
mysqlbinlog --start-datetime="2025-06-23 10:00:00" \
            --stop-datetime="2025-06-23 10:29:00" \
            /var/log/mysql/mysql-bin.000123 > binlog_recovery.sql

This will extract all queries between 10:00 and 10:29 AM before the DROP TABLE happened

2. 🔍 Open the file and check queries
``` vim binlog_recovery.sql ```
Look for:
- CREATE TABLE reporting_data ...
- INSERT INTO reporting_data ...
- Stop before the DROP TABLE command

3. ✂️ Remove anything after DROP TABLE (if present)
Only keep CREATE, INSERT, ALTER — discard destructive commands.

4. 🧪 Restore on a safe test database first
mysql -u root -p test_recovery_db < binlog_recovery.sql

5. 🚀 Apply to production
Once validated:
mysql -u root -p my_database < binlog_recovery.sql

💡 Real-Life Tip:
-------------------
In my case, I used:
 - mysqlbinlog to extract queries
 - A timestamp 2 minutes before the accidental drop
 - Validated recovery on a staging server
 - Then restored just that table on prod
➡️ We recovered 100% of the data with zero downtime for other tables.

--------------------------------------------------------------------

Q: A table was dropped in PostgreSQL production. How do you recover it?
- PostgreSQL doesn’t keep binary logs you cant replay like MySQL, so I rely on Point-In-Time Recovery (PITR) with archived WAL files.

-  Steps I follow:
-------------------

1- Verify backups & WAL archiving: Make sure we have a recent base backup and continuous WAL archives.

2- Spin up a temporary restore instance:
   * Restore the latest base backup to another server or a new data directory.
   * In recovery.conf (or postgresql.conf on newer versions) set recovery_target_time just before the DROP happened.

3-  Start PostgreSQL in recovery mode: It replays WAL until the target time, recreating the table and all data up to that point.

4- Dump only the rescued table:
> pg_dump -Fc -t public.reporting_data recovered_db > reporting_data.dump

5- Restore into production:
> pg_restore -d prod_db -Fc reporting_data.dump

6- Validate rows and indexes, then drop the temporary server.

```

---

### 9. **A user is seeing “Too many connections” error – what do you do?**

* Check `max_connections` in MySQL config
* Use:

  ```sql
  SHOW STATUS LIKE 'Threads_connected';
  ```
* Kill inactive connections or increase limit cautiously.

---

### 10. **How to migrate a MySQL database to another server?**

1. Dump the DB:

   ```bash
mysqldump -h <HOST_OR_ENDPOINT> -u <USERNAME> -p <DATABASE_NAME> > dump.sql
   ```
2. Transfer:

   ```bash
   scp dump.sql user@server:/path/
   ```
3. Import on new server:

   ```bash
   mysql -u user -p dbname < dump.sql
   ```

---

## 🔹 **Advanced (for scaling or high availability)**

### 11. **Explain replication in MySQL.**

* Master-slave setup:

  * Master handles writes
  * Slaves replicate data and serve read traffic
* Use `binlog` and replication user
* Helps with read-scaling and backup

---

### 12. **What are binlogs used for?**

* To log all changes to the database
* Useful for replication and point-in-time recovery

---

### 13. **What is a deadlock? How do you detect and prevent it?**

* A deadlock happens when two or more transactions block each other, waiting for locks the other holds — so none can proceed.
* exp scenario would be if Transaction A locked row 1, Transaction B locked row 2, and both tried to update the other’s row — causing a deadlock.
* 🔍 **How I detect it:**
 
* In MySQL, I check:
```
SHOW ENGINE INNODB STATUS;
```
→ It shows the last deadlock details.
  
* In PostgreSQL, I check logs: 
``` 
  ERROR: deadlock detected
  DETAIL: Process 1234 waits for ShareLock...
```

* 🛡️ How I prevent it:
  * Always lock rows in a consistent order across code paths
  * Keep transactions short and avoid holding locks too long
  * Add timeouts or retries for retry-safe transactions
  * In high-contention cases, use optimistic locking (version check)


# ===================================================================================

# Scenario Base


### 🔧 **1. Scenario: Replica is Missing 2 Hours of Data**

**🧩 Situation:**
Your MySQL replica server somehow missed 2 hours of data due to replication lag, crash, or network failure.

**✅ Solution:**

```
1- VIA BINLOGS
----------------

You can recover missing data on a MySQL replica using binary logs, without doing a full dump — if:
- Binary logs (log_bin) are enabled on the master
- The master still retains the logs for the time the replica missed (not purged)

🔄 How recovery using binary logs works:
-----------------------------------------
🧭 Scenario: Your replica crashed or was offline from 2 PM to 4 PM. Now it's back up and you're missing changes from that 2-hour window

# ✅ Steps to Recover Replica Using Binlogs
-------------------------------------------

1- Check current replica status: SHOW SLAVE STATUS\G
Note the current "Relay_Log_Pos" and "Master_Log_File" to identify what’s missing.

2- On the master, identify binlogs covering 2–4 PM:
mysqlbinlog --start-datetime="2025-06-23 14:00:00" \
            --stop-datetime="2025-06-23 16:00:00" \
            /var/lib/mysql/mysql-bin.000123 > missing.sql

3- Copy the SQL to the replica and replay it:
scp missing.sql replica:/tmp/
mysql -u root -p < /tmp/missing.sql
This replays only the missing data.

4- Restart replication:
START SLAVE;
```

```
2- VIA MYSQL DUMP 
------------------

🧭 Scenario: Your replica missed 2 hours of data. You want to resync it cleanly using mysqldump.

🔧 Steps (using mysqldump):
-----------------------------

🖥️ On the master:
-------------------
1- Take a consistent dump with binlog coordinates:

mysqldump -u root -p \
  --single-transaction \
  --master-data=2 \
  --databases your_db_name > dump.sql

"--master-data=2" adds a comment like: "-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000123', MASTER_LOG_POS=456789;"

2- 🚚 Transfer to replica:
scp dump.sql replica:/tmp/

🖥️ On the replica:
--------------------
1- Stop replication: STOP SLAVE;

2- Restore the dump: 
mysql -u root -p your_db_name < /tmp/dump.sql

3-Configure replication to resume from correct log/position:

CHANGE MASTER TO
  MASTER_LOG_FILE='mysql-bin.000123',
  MASTER_LOG_POS=456789;

4- Start slave:

START SLAVE;

5-Monitor:

SHOW SLAVE STATUS\G

```

```
 
 3. AWS RDS (MySQL)
 ----------------
- Create a fresh replica from the master and point the app to it.

🟢 Why it works well:
- AWS handles replication setup and data sync automatically
- No need for mysqldump, binlog recovery, or manual CHANGE MASTER TO
- Fast and safe for production use
- The old replica can be deleted afterward

❗️Q: What if the RDS master got corrupted and the last 2 hours of data is missing? What can you do?

✅ Two options depending on backup setup:
-------------------------------------------

1️⃣ Point-in-Time Recovery (PITR) — if automated backups & WAL logs are enabled
--------------------------------------------------------------------------------
* 🧭 Goal: Recover master to a time just before corruption

* 🔧 Steps:
---------------
1- Go to RDS Console → Snapshots → Automated backups
2- Click “Restore to point in time”
3- Choose a timestamp (e.g., 1 minute before corruption — say 12:58 PM)
4- AWS creates a new RDS instance with all data up to that moment
5- Promote it to replace the old master, or:
6- Compare/reconcile data from new instance into live prod (if partial loss)
7- Re-point apps or replicas as needed

 master got corrupted after a faulty schema change at 1 PM. We restored a new instance from PITR at 12:59 PM and redirected traffic there. We lost only a few seconds of writes.

2️⃣ Use Read Replica (if it had the missing data)
-------------------------------------------------
If your replica is still healthy and ahead of master, you can:
1- Promote the replica
2- It becomes a standalone writable instance
3- Point app to this new DB
4- This avoids the 2-hour gap on master

```

---

### 🔄 **2. Scenario: Sync Only Specific Tables Between Databases**

```
🎯 Use Case:
----------------
You want to sync only a few selected tables (not the whole DB) from:

* source_db on Server A → target_db on Server B
This is useful when:
* You want to migrate only critical data
* You’re syncing staging/prod selectively
* Full DB size is too large

✅ Method 1: Using mysqldump for Selected Tables
------------------------------------------------

* On Source Server:
------------------
mysqldump -u root -p source_db table1 table2 > selected_tables.sql

This exports only table1 and table2.

* Transfer file to target:  
scp selected_tables.sql user@target-server:/tmp/

* On Target Server:
--------------------
mysql -u root -p target_db < /tmp/selected_tables.sql

✅ Tables are now synced.


✅ Method 2: Use REPLICATION FILTER for Replicating Only Certain Tables (Advanced)
---------------------------------------------------------------------------

If setting up replication, but want only a few tables:

# In my.cnf on replica
replicate-do-db = source_db
replicate-do-table = source_db.table1
replicate-do-table = source_db.table2

This ensures only selected tables are replicated.

⚠️ Works only if master is dedicated to those tables; replicate-do-* has edge cases.

✅ Method 3: Use pt-table-sync from Percona Toolkit
----------------------------------------------------
* Great when tables already exist on both sides
* Can sync differences row-by-row
pt-table-sync --execute \
  --sync-to-master h=target-server,D=target_db,t=table1,u=user,p=pass \
  h=source-server,D=source_db,t=table1

✅ Method 4: Use DMS
---------------------
AWS DMS (Database Migration Service) is a perfect fit for syncing specific tables between databases — especially in AWS environments.

Why Use DMS for Selective Table Sync?
-----------------------------------------
* No downtime (CDC – Change Data Capture)
* Can replicate specific tables only
* Works across:
    * On-prem → RDS
    * RDS → RDS
    * MySQL → PostgreSQL
* Can do ongoing replication or one-time migration

🛠️ Steps to Sync Selected Tables Using DMS:
-------------------------------------------
1. Create a replication instance
  * Go to AWS DMS console → Replication Instances
  * Choose instance size (start with dms.t3.medium for small loads)
  * Place it in same VPC/security group as your source/target DBs

2. Set up source and target endpoints
  * Source = your MySQL (can be on-prem, EC2, or RDS)
  * Target = your destination MySQL
  * Test connection from DMS

3. Create a migration task
* Choose Table mapping:
   * Specify schema and tables like:
 {
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "1",
      "object-locator": {
        "schema-name": "mydb",
        "table-name": "users"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "selection",
      "rule-id": "2",
      "rule-name": "2",
      "object-locator": {
        "schema-name": "mydb",
        "table-name": "orders"
      },
      "rule-action": "include"
    }
  ]
}

* Choose Migration type:
   * ✅ "Migrate existing data and replicate ongoing changes" — for continuous sync
   * Or "Migrate existing data only" — for one-time copy

* 4. Run the task and monitor
  * View progress in DMS console
  * Check table-level migration status
  * DMS will copy schema and data only for specified tables

🧠 Real-Life Example:
----------------------
We had to sync just customer, invoice, and payments tables from an on-prem MySQL to AWS RDS MySQL nightly.
Instead of building a cron job or managing mysqldump, we used AWS DMS with selective table mapping and ongoing replication. It worked flawlessly with zero manual effort.
```

---

### 3. Scenario: Schema Changed on Master, Not Replica

```
🧩 Issue: A developer added a column on the master, but replica wasn't updated. Now replication is broken with errors like
"Error: Column count doesn’t match value count at row X"

✅ Why this happens:
----------------------
- MySQL row-based replication depends on schema consistency between master and replica
- If schema differs, even valid data insertions can fail on replica

🛠️ Steps to Fix (Safely):
---------------------------

🔍 1. Identify the schema difference
On master:
----------
SHOW CREATE TABLE your_table;

Compare with replica:
-----------------------
SHOW CREATE TABLE your_table;


🛑 2. Stop replication on replica temporarily
STOP SLAVE;


🧱 3. Manually apply the schema change

If the master added:
-----------------------
ALTER TABLE your_table ADD COLUMN status VARCHAR(20) DEFAULT 'active';

Apply exact same command on replica:
------------------------------------
ALTER TABLE your_table ADD COLUMN status VARCHAR(20) DEFAULT 'active';

Ensure:
----------
Same column name, type, position, default

✅ 4. Restart replication
START SLAVE;

Check:
------
SHOW SLAVE STATUS\G

Make sure:
------------
* Slave_SQL_Running: Yes
* Seconds_Behind_Master: 0

✅ Best Practices to Avoid This:

| Practice                               | Description                               |
| -------------------------------------- | ----------------------------------------- |
| Use Infrastructure as Code (IaC)       | Automate schema changes                   |
| Use tools like Liquibase/Flyway        | Schema changes go through version control |
| Always test schema changes on staging  | Ensure replication is not broken          |
| Keep master and replica in schema sync | Required for row-based replication        |


🧠 Real-Life Example:
---------------------
A teammate ran an ALTER TABLE in prod without updating staging (replica). Replication stopped with a column mismatch error.
I paused the slave, applied the exact ALTER TABLE on the replica, restarted the slave, and replication caught up in 1 minute.

```

---

### 🐌 4. Scenario: Replica Lagging Behind Master

**🧩 Symptoms:**

```sql
SHOW SLAVE STATUS\G
```
* `Seconds_Behind_Master`: shows lag (e.g., 7200 seconds)
* `Slave_SQL_Running`: No
* `Last_SQL_Error`: may show slow query

```
✅ Why This Happens:
-----------------------
* Replica SQL thread hit a long-running query, blocking others
* Insufficient resources (CPU/disk I/O) on replica
* Locks or deadlocks delaying replication
* Large transactions or schema mismatches

🛠️ How to Fix It:
-----------------

1️⃣ Check the error
---------------------
SHOW SLAVE STATUS\G

Look at Last_SQL_Error — it will often point to:
  * A failed ALTER TABLE
  * Data type mismatch
  * Or a very slow INSERT/UPDATE/DELETE

2️⃣ Fix the root cause
----------------------
- If it’s a bad query (e.g., large ALTER TABLE), you can:
   - Fix schema manually on replica
   - Skip the event (use with caution):  
                 SET GLOBAL sql_slave_skip_counter = 1;
                 START SLAVE;
- If it’s resource-related:
     * Check CPU, I/O, disk space
     * Kill other heavy processes
     * Add replicas with higher specs

3️⃣ Monitor replication catch-up
---------------------------------
SHOW SLAVE STATUS\G

* Seconds_Behind_Master should decrease steadily
* Once it reaches 0, replica is in sync

🧠 Real-Life Example:
---------------------
In one case, our replica lagged by 2 hours because a DELETE with no index locked millions of rows.
I optimized the query on the master, killed the lagging thread, and added an index. Replica caught up in 5 minutes after that.
```

---

### 📊 5. Scenario: Staging and Production DBs Have Diverged

**🧩 Need:** Reconcile them without full overwrite.

```
✅ Fix using Percona tools:

pt-table-checksum --nocheck-replication-filters --replicate=db.checksums
pt-table-sync --execute h=prod_host,D=db h=staging_host,D=db


* This computes row-level checksums.
* Only syncs mismatched rows.
* Great for detecting silent corruption or schema drift.
```

---

### 🌐 6. Scenario: Keep a Live Read-Only Copy in Another Region

**🧩 Need:** You want high availability and near real-time sync by maintaining a read-only replica of your production DB in a different AWS region — for disaster recovery or global reads.

```
🔁 Option 1: AWS RDS Read Replica Across Regions
------------------------------------------------
If you’re using Amazon RDS for MySQL or PostgreSQL, the easiest and most reliable method is:

🛠️ Steps:
------------
1. Go to RDS Console
2. Select your primary DB instance
3. Click “Create Read Replica”
4. Choose a different region (e.g., us-west-1 if your master is in ap-south-1)
5. AWS handles:
   * Cross-region replication
   * Read-only setup
   * Auto data sync via async replication
📌 The replica is read-only and can be promoted during disaster.

✅ Pros:
--------
| Feature           | Benefit                         |
| ----------------- | ------------------------------- |
| Fully managed     | No manual replication setup     |
| Automatic syncing | Near real-time lag (\~seconds)  |
| Scalable          | Add more read replicas later    |
| DR-ready          | Can promote to master if needed |

🔒 Requirements:
-----------------
- RDS must allow cross-region replication (enabled by default)
- Replica instance type and storage must match or be compatible

compatible

🔁 Option 2: Manual Replication (Self-Managed MySQL/PostgreSQL)
--------------------------------------------------------------
If you're not using RDS, or want full control:

- Set up a replica manually in another region:
   * Use mysqldump + CHANGE MASTER TO or GTID-based setup
   * Open ports (e.g., 3306), configure security groups and VPC peering
   * Use SSL/TLS for secure cross-region transfer

⚠️ More effort, but gives more control

🧠 Real-Life Example:
---------------------------
We had a critical production DB in ap-south-1, and the client required a read-only backup in us-east-1 for DR and BI queries.
We created a cross-region RDS read replica, monitored lag via CloudWatch, and tested failover monthly. During a region outage simulation, we promoted the replica and switched traffic in 5 minutes.

```


