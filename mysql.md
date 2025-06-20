
Here’s a list of **common and scenario-based MySQL interview questions** (especially for someone with \~1 year of DevOps or backend experience), along with brief **explanations/answers** so you can prep easily:

---

## 🔹 **Basic to Intermediate MySQL Questions**

### 1. **What is the difference between `CHAR` and `VARCHAR`?**

* `CHAR`: Fixed-length (faster for small strings).
* `VARCHAR`: Variable-length (space-efficient for long/irregular strings).

---

### 2. **How do you perform a backup of a MySQL database?**

* Using `mysqldump`:

  ```bash
  mysqldump -u user -p database_name > backup.sql
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

* Restore from `mysqldump` backup or point-in-time recovery using binlogs (if enabled).

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
   mysqldump -u user -p dbname > dump.sql
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

* Two transactions waiting on each other’s lock.
* Detect using:

  ```sql
  SHOW ENGINE INNODB STATUS;
  ```
* Avoid long transactions, consistent locking order.

# ===================================================================================

# Scenario Base

Here’s an **expanded explanation with detailed step-by-step reasoning** for each **MySQL scenario-based interview question**, especially focused on **data recovery, sync, and troubleshooting** — perfect for someone with 1–2 years of DevOps or backend experience.

---

### 🔧 **1. Scenario: Replica is Missing 2 Hours of Data**

**🧩 Situation:**
Your MySQL replica server somehow missed 2 hours of data due to replication lag, crash, or network failure.

**✅ Solution:**

#### ➤ Step 1: Check if binary logs (`binlogs`) are enabled on the master:

```sql
SHOW MASTER STATUS;
```

This gives:

* `File`: e.g., `mysql-bin.000123`
* `Position`: e.g., `456789`

#### ➤ Step 2: Check the replication status on the replica:

```sql
SHOW SLAVE STATUS\G
```

Key fields:

* `Seconds_Behind_Master`
* `Last_SQL_Error`
* `Relay_Log_Pos`

#### ➤ Step 3: If replication is broken, restart replication:

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_LOG_FILE='mysql-bin.000123',
  MASTER_LOG_POS=456789;
START SLAVE;
```

#### ➤ If binlogs are not enough (e.g., purged), and you know the time gap:

* Export missing rows:

```sql
mysqldump -u user -p dbname table_name --where="created_at BETWEEN '2025-06-17 12:00:00' AND '2025-06-17 14:00:00'" > delta.sql
```

* Import into replica:

```bash
mysql -u user -p dbname < delta.sql
```

🧠 **Why it matters:** This avoids downtime and a full DB restore.

---

### 🧨 **2. Scenario: Table Accidentally Dropped**

**🧩 Situation:** Someone dropped a table in production and you don’t have a logical backup (.sql).

**✅ Solution:**

#### A. Binlogs enabled? → Use `mysqlbinlog` to recover it:

```bash
mysqlbinlog --start-datetime="2025-06-17 10:00:00" /var/lib/mysql/mysql-bin.000012 > recovery.sql
```

* Then search for `DROP TABLE` and recover the previous state.

#### B. If using AWS RDS:

* Use **point-in-time restore**:

  * Restore RDS to a new instance up to a specific time.
  * Export the missing table.
  * Import into current prod.

---

### 🔄 **3. Scenario: Sync Only Specific Tables Between Databases**

**🧩 Situation:** You want to copy only one or two tables from Prod → Staging.

**✅ Options:**

#### A. Using `mysqldump`:

```bash
mysqldump -u user -p db_name table1 table2 > table_backup.sql
mysql -u user -p target_db < table_backup.sql
```

#### B. Using Percona Toolkit (real-time delta sync):

```bash
pt-table-sync --execute h=source_host,D=db,t=table1 h=target_host,D=db,t=table1
```

📌 Use when:

* Data has diverged
* Avoiding downtime
* You want real-time sync (for large tables)

---

### ⚠️ **4. Scenario: Schema Changed on Master, Not Replica**

**🧩 Issue:** Developer added a column on master, but not on replica.

**✅ Fix:**

* Replica throws: `Column count doesn't match value count`.
* You must:

```sql
STOP SLAVE;
ALTER TABLE table_name ADD COLUMN new_column ...;
START SLAVE;
```

---

### 🐌 **5. Scenario: Replica Lagging Behind Master**

**🧩 Symptoms:**

```sql
SHOW SLAVE STATUS\G
```

* `Seconds_Behind_Master`: shows lag (e.g., 7200 seconds)
* `Slave_SQL_Running`: No
* `Last_SQL_Error`: may show slow query

**✅ Fix:**

* Check for long-running queries on replica.
* Increase `innodb_buffer_pool_size` if memory is low.
* Tune indexes on replica.
* For extreme lag, restart replication from latest position (if safe).

---

### 📊 **6. Scenario: Staging and Production DBs Have Diverged**

**🧩 Need:** Reconcile them without full overwrite.

**✅ Fix using Percona tools:**

```bash
pt-table-checksum --nocheck-replication-filters --replicate=db.checksums
pt-table-sync --execute h=prod_host,D=db h=staging_host,D=db
```

* This computes row-level checksums.
* Only syncs mismatched rows.
* Great for detecting silent corruption or schema drift.

---

### 🌐 **7. Scenario: Keep a Live Read-Only Copy in Another Region**

**🧩 Need:** High availability with near-real-time sync.

**✅ Solution:**

* Set up **asynchronous replication** using:

  ```sql
  CHANGE MASTER TO ...;
  START SLAVE;
  ```
* Or use **GTID-based replication** (safer crash recovery).
* If using RDS:

  * Use **Read Replica**
  * Or **Cross-Region Replica**

---

## 🔁 Summary Table

| Scenario                       | Best Tool/Command                        | Use Case                           |
| ------------------------------ | ---------------------------------------- | ---------------------------------- |
| Missing 2 hours of data        | `mysqlbinlog`, `mysqldump`, replication  | Partial restore                    |
| Table deleted                  | `mysqlbinlog` or point-in-time recovery  | Recover accidentally dropped table |
| Sync specific tables           | `mysqldump`, `pt-table-sync`             | Partial sync between DBs           |
| Schema mismatch in replication | `ALTER TABLE` on replica                 | Prevent replication failure        |
| Replica lag                    | `SHOW SLAVE STATUS`, tuning, or reconfig | Ensure real-time replication       |
| Diverged DBs                   | `pt-table-checksum`, `pt-table-sync`     | Reconcile two envs                 |
| Real-time live copy            | Asynchronous / GTID replication          | Disaster recovery                  |

---

Would you like a downloadable PDF version or an Excel tracker of these scenarios and resolutions?
