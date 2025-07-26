Absolutely, Tarun. Below is the **complete real-world Point-in-Time Recovery (PITR) setup using Percona XtraBackup** — with **full exact commands including `sudo`** where needed. You can copy-paste and run this on your Ubuntu production or staging environment.

---

## 🧨 SCENARIO RECAP (For Reference):

* 🔁 Daily **XtraBackup** runs at `1:00 AM`
* ❌ User **accidentally deletes rows** at `12:00 PM`
* 🛠️ You need to restore database to **11:59:59 AM** using:

  * ✅ Full backup
  * ✅ Binary logs

---

## ✅ STEP-BY-STEP RESTORE WITH REAL COMMANDS (Full Format)

---

### ⚙️ 0. **Ensure Prerequisites are Set (Only Once)**

#### A. Enable Binary Logging

Edit MySQL config:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add or ensure:

```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_format=ROW
```

Restart MySQL:

```bash
sudo systemctl restart mysql
```

---

### ⏱️ 1. Take Full Backup at 1:00 AM

```bash
sudo xtrabackup --backup \
  --target-dir=/backups/full-1am \
  --user=root \
  --password='your_password'
```

---

### 🧹 2. Prepare the Backup

```bash
sudo xtrabackup --prepare \
  --target-dir=/backups/full-1am
```

---

### 🚨 3. Data Deletion Happens at 12:00 PM

(For testing simulation — this is what happens accidentally:)

```sql
DELETE FROM transactions WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

---

### 🛑 4. Stop MySQL and Clean Data Folder

```bash
sudo systemctl stop mysql
sudo mv /var/lib/mysql /var/lib/mysql_broken_$(date +%F-%T)
sudo mkdir /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
```

---

### 📦 5. Restore Backup to MySQL Data Directory

```bash
sudo xtrabackup --copy-back \
  --target-dir=/backups/full-1am
```

Fix permissions:

```bash
sudo chown -R mysql:mysql /var/lib/mysql
```

---

### 🔎 6. Get Binlog Position from Backup

```bash
cat /backups/full-1am/xtrabackup_binlog_info
```

Example output:

```
mysql-bin.000123  48532
```

---

### 📥 7. Extract Transactions From 1 AM → 11:59:59 AM

```bash
sudo mysqlbinlog \
  --start-position=48532 \
  --stop-datetime="2025-07-25 11:59:59" \
  /var/lib/mysql/mysql-bin.000123 > /tmp/pitr_recovery.sql
```

> 🔁 Adjust `/var/lib/mysql/mysql-bin.000123` if your binlogs are stored elsewhere. Use `ls /var/lib/mysql` to check.

---

### 🚀 8. Start MySQL Server

```bash
sudo systemctl start mysql
```

---

### 🧪 9. Replay Safe SQL (Restoring Data From Binlogs)

```bash
mysql -u root -p < /tmp/pitr_recovery.sql
```

Enter your MySQL password when prompted.

---

### ✅ 10. Confirm Final Result

Query your table:

```sql
SELECT COUNT(*) FROM transactions;
SELECT * FROM transactions ORDER BY created_at DESC LIMIT 10;
```

---

## ✅ FINAL CHECKLIST (QUICK REFERENCE)

| Task          | Command                                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Backup        | `sudo xtrabackup --backup --target-dir=/backups/full-1am --user=root --password='your_password'`                         |
| Prepare       | `sudo xtrabackup --prepare --target-dir=/backups/full-1am`                                                               |
| Stop MySQL    | `sudo systemctl stop mysql`                                                                                              |
| Restore       | `sudo xtrabackup --copy-back --target-dir=/backups/full-1am`                                                             |
| Fix Ownership | `sudo chown -R mysql:mysql /var/lib/mysql`                                                                               |
| Start MySQL   | `sudo systemctl start mysql`                                                                                             |
| Extract SQL   | `sudo mysqlbinlog --start-position=48532 --stop-datetime="YYYY-MM-DD HH:MM:SS" /path/to/binlog > /tmp/pitr_recovery.sql` |
| Replay SQL    | `mysql -u root -p < /tmp/pitr_recovery.sql`                                                                              |

---

## 💡 Tips:

* Always double-check the **binlog filename** and **start-position** from `xtrabackup_binlog_info`
* You can **redirect binlogs** to a safe location via `log-bin=/logs/binlog` in config
* Ensure **enough disk space** exists before restoring large backups

---

Would you like me to:

* 🔄 Create a **bash script** for full PITR automation?
* 🧪 Create a **test lab in Docker** where you simulate all of this live?

Let me know, Tarun — you're seriously operating like a production DBA now.
