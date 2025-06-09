# Oracle Data Guard 19c Setup with Data Guard Broker
- Standby Database with Different File Paths :

In this setup, the standby database uses different file locations than the primary database. This is useful when the folder structure or disk mount points on the standby server are not the same as on the primary.

To handle this, we use these RMAN parameters during standby creation:

  -  **DB_FILE_NAME_CONVERT**: Converts the primary datafile paths to the standby paths.
  -  **LOG_FILE_NAME_CONVERT**: Converts the primary log file paths to the standby paths.
  -  CONTROL_FILES, DB_CREATE_FILE_DEST, DB_RECOVERY_FILE_DEST: Set the location for control files, datafiles, and archived logs on the standby server.
  -  LISTENER1528 on port **1528** is configured and used for the Data Guard Broker service (**mydb_DGMGRL**).

These settings allow Oracle to correctly place files during duplication and redo apply, even if the standby has a different directory layout. 

**Author**: *Zubair OCP*
**Date**: *2025-06-09*

---

## 1. Listener Configuration (Primary & Standby)

Edit `$ORACLE_HOME/network/admin/listener.ora` to define two listeners on both primary and standby hosts:

```
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = zubair)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

USE_SID_AS_SERVICE_LISTENER = ON

LISTENER1528 =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = zubair)(PORT = 1528))
    )
  )

SID_LIST_LISTENER1528 =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = mydb_DGMGRL)
      (ORACLE_HOME = /oracle/product/19)
      (SID_NAME = mydb)
    )
  )
```

## 2. Configure TNSNAMES

Edit `$ORACLE_HOME/network/admin/tnsnames.ora`:

```
LISTENER_ALL =
  (ADDRESS_LIST =
    (ADDRESS=(PROTOCOL=TCP)(HOST=zubair)(PORT=1521))
    (ADDRESS=(PROTOCOL=TCP)(HOST=zubair)(PORT=1528))
  )
```

## 3. Set LOCAL\_LISTENER

```sql
ALTER SYSTEM SET LOCAL_LISTENER='LISTENER_ALL' SCOPE=BOTH;
ALTER SYSTEM REGISTER;
```

Check status:

```bash
lsnrctl status
lsnrctl status LISTENER1528
```

---

## 4. Enable Archivelog Mode on Primary

```sql
sqlplus / as sysdba
ARCHIVE LOG LIST;
ALTER SYSTEM SET log_archive_dest_1='LOCATION=/oracle/archivelog' SCOPE=SPFILE;
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ARCHIVE LOG LIST;

ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO;
```

---

## 5. Prepare Standby Directory Structure

```bash
mkdir -p /oracle/product/db/orabase/oradata/mydb_stb/
mkdir -p /oracle/product/db/orabase/oradata/mydb_stb/onlinelog/
mkdir -p /oracle/product/db/orabase/oradata/mydb_stb/controlfile/
mkdir -p /oracle/product/db/orabase/fast_recovery_area/MYDB_STB/
mkdir -p /oracle/product/db/orabase/admin/mydb/adump/

chown -R oracle:oinstall /oracle/product/db/orabase/oradata/mydb_stb/
chown -R oracle:oinstall /oracle/product/db/orabase/fast_recovery_area/MYDB_STB/
chown -R oracle:oinstall /oracle/product/db/orabase/admin/mydb/adump/
chmod -R 750 /oracle/product/db/orabase/oradata/mydb_stb/
chmod -R 750 /oracle/product/db/orabase/fast_recovery_area/MYDB_STB/
chmod -R 750 /oracle/product/db/orabase/admin/mydb/adump/
```

---

## 6. Standby Server Setup

### Prepare for Duplicate

Create a parameter file for the standby database:

```bash
echo "*.db_name='mydb'" > /oracle/initstb.ora
```

Start the auxiliary instance on the standby server:

```bash
export ORACLE_SID=cdb1
sqlplus / as sysdba

SQL> STARTUP NOMOUNT PFILE='/oracle/initstb.ora';
```

---

## 7. Duplicate Standby Database from Active Primary

```bash
rman TARGET sys/oracle123@mydb_prd AUXILIARY sys/oracle123@mydb_stb
```

```rman
DUPLICATE TARGET DATABASE FOR STANDBY
FROM ACTIVE DATABASE
DORECOVER
SPFILE
  SET DB_UNIQUE_NAME='mydb_stb'
  SET LOCAL_LISTENER='(ADDRESS=(PROTOCOL=TCP)(HOST=zubair-standby)(PORT=1528))'
  SET LOG_ARCHIVE_DEST_1='LOCATION=USE_DB_RECOVERY_FILE_DEST'
  SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(mydb,mydb_stb)'
  SET CONTROL_FILES='/oracle/product/db/orabase/oradata/mydb_stb/controlfile/control01.ctl'
  SET DB_FILE_NAME_CONVERT='/oracle/product/orabase/oradata/mydb/', '/oracle/product/db/orabase/oradata/mydb_stb/'
  SET LOG_FILE_NAME_CONVERT='/oracle/product/orabase/oradata/mydb/onlinelog/', '/oracle/product/db/orabase/oradata/mydb_stb/onlinelog/'
  SET DB_RECOVERY_FILE_DEST='/oracle/product/db/orabase/fast_recovery_area/MYDB_STB'
  SET DB_CREATE_FILE_DEST='/oracle/product/db/orabase/oradata/mydb_stb/'
  SET AUDIT_FILE_DEST='/oracle/product/db/orabase/admin/mydb/adump'
  SET FAL_SERVER='mydb'
  SET FAL_CLIENT='mydb_stb'
  SET JOB_QUEUE_PROCESSES='0'
  COMMENT 'Is standby'
  NOFILENAMECHECK;
```

---

## 8. Enable DG Broker Configuration

On both primary and standby:

```sql
ALTER SYSTEM SET DG_BROKER_START=TRUE SCOPE=BOTH;
```

Create DG configuration:

```bash
dgmgrl sys/oracle123@mydb

CREATE CONFIGURATION dgconfig AS
  PRIMARY DATABASE IS mydb
  CONNECT IDENTIFIER IS mydb;

ADD DATABASE mydb_stb AS
  CONNECT IDENTIFIER IS mydb_stb
  MAINTAINED AS PHYSICAL;

ENABLE CONFIGURATION;
```

Check status:

```bash
SHOW CONFIGURATION;
SHOW DATABASE VERBOSE mydb;
SHOW DATABASE VERBOSE mydb_stb;
```

## 9. Check Redo Apply Status (Standby)

```sql
SELECT PROCESS, STATUS, THREAD#, SEQUENCE# 
FROM V$MANAGED_STANDBY 
WHERE PROCESS LIKE 'MRP%';
```

* `MRP0` with status `APPLYING_LOG` indicates apply is running.
* If missing or status is `WAITING_FOR_LOG`, check transport or gap issues.

---

## 10. Switchover & Failover (Optional)

Switchover:

```bash
DGMGRL> SWITCHOVER TO mydb_stb;
```

Failover:

```bash
DGMGRL> FAILOVER TO mydb_stb;
```

---

## 11. Troubleshooting: Database Not Syncing or Gap Detected

### a. Check DG Broker Configuration

```bash
dgmgrl> show configuration;
dgmgrl> show database verbose mydb_stb;
```

### b. Check Alert Log & Standby Log

```bash
tail -f $ORACLE_BASE/diag/rdbms/*/trace/alert_*.log
```

Check for errors like:

* ORA-16810
* ORA-16724

### c. Check Gap & Recovery Status

```sql
SELECT DEST_ID, STATUS, ERROR FROM V$ARCHIVE_DEST;
SELECT THREAD#, LOW_SEQUENCE#, HIGH_SEQUENCE# FROM V$ARCHIVE_GAP;
```

### d. Manual Archive Log Shipping

```bash
scp /oracle/archivelog/1_12345_*.arc standby:/oracle/archivelog/
```

```sql
ALTER DATABASE REGISTER LOGFILE '/oracle/archivelog/1_12345_*.arc';
RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

### e. FAL\_SERVER / FAL\_CLIENT Check

```sql
SHOW PARAMETER fal;
```

### f. Restart MRP

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

### g. Network Check

```bash
lsnrctl status LISTENER1528
tnsping mydb_stb
```

---

This document is a simplified Oracle 19c Data Guard setup using Data Guard Broker with different file paths on standby and a dedicated listener on port 1528 for DG services.
