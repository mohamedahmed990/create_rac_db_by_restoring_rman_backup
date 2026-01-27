# Solution Action Plan

## Purpose
Restore a RAC database backup from one 2-node cluster to another 2-node cluster with identical ASM layout (+DATA, +FRA, +CRS) and create the database on the empty target cluster.

---

## Prerequisites on the Target Cluster
- Grid Infrastructure installed, configured, and healthy (Clusterware up)
- ASM running with required disk groups mounted: +DATA and +FRA (CRS exists but is not used for database files)
- RDBMS home installed at the same version and patched to the same RU level as source
- Networking, SCAN, VIPs, passwordless SSH, OS groups/users aligned per standards
- Sufficient space in +DATA and +FRA

---

## High-Level Approach
1. Take an RMAN backup (database + archivelogs + controlfile autobackup) on the source cluster
2. Transfer backups to the target cluster
3. Build a single parameter file on the target suitable for RAC (cluster_database=TRUE)
4. Use RMAN to restore and recover to +DATA/+FRA
5. Register and configure the RAC database resource with srvctl
6. Create instances on both nodes

---

## Detailed Steps

### A. On the Source Cluster: Create a Complete RMAN Backup
1. **Enable CONTROLFILE AUTOBACKUP**  
   ```
   RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
   ```

2. **Take Backup to a Filesystem or NFS Share**  
   ```
   RMAN> run {
       allocate channel c1 type disk format '/backup/%U';
       backup database plus archivelog;
   }
   ```

3. **Verify Backup and Note Controlfile Autobackup Path**  
   - Confirm backup completion  
   - Record controlfile autobackup piece location

---

### B. Transfer Backups to the Target Cluster
- Copy all backup pieces (including controlfile autobackup and SPFILE autobackup if created) to a target directory (e.g., `/backup`)  
- One node access is sufficient for restore; ensure both nodes can access if needed

---

### C. Prepare the Target RAC Database Initialization Parameters
1. **Create Minimal PFILE on Target for NOMOUNT Startup**  
   File: `/tmp/initDBNAME.ora`  
   Example Contents:
   ```
   db_name=DBNAME
   db_unique_name=DBNAME
   diagnostic_dest=/u01/app/oracle
   memory_target=8G
   control_files='+FRA/DBNAME/CONTROLFILE/control01.ctl'
   cluster_database=true
   # Instance-specific undo and redo will be set later; keep minimal for now
   ```
   *Adjust parameters to match your environment*

---

### D. Restore Controlfile and Mount Database (Target Cluster)

1. **Start Instance NOMOUNT Using PFILE**  
   ```
   sqlplus / as sysdba
   ```
   ```
   startup nomount pfile='/tmp/initDBNAME.ora';
   ```

2. **Restore Controlfile from Autobackup**  
   ```bash
   rman target /
   ```

   ```
   restore controlfile from '/backup/<controlfile_autobackup_piece>';
   alter database mount;
   ```
   - Use DBID if autobackup path unknown; default search applies

3. **Catalog Backups if Path Differs**  
   ```
   catalog start with '/backup/' noprompt;
   ```

---

### E. Restore and Recover Database into ASM
- Datafiles and tempfiles typically go to +DATA  
- FRA/log_archive_dest typically goes to +FRA  

**Example RMAN Run Block**  

```bash
RMAN> run {
    set newname for database to '+DATA';
    restore database;
    switch datafile all;
    recover database;
}
```
*Note: For point-in-time recovery, use `SET UNTIL {SCN|TIME|SEQUENCE}` before restore*

---

### F. Prepare Redo Logs and Instance/RAC Settings
1. **Create/Relocate Online Redo Logs**  
   - Place in +DATA (or +FRA per policy)  
   - Create threads for each RAC instance (example for two instances):
     
   ```bash
   sqlplus / as sysdba
   ```
   
   -- Option A: Rename (if files exist and need moving)
   
   ```
   alter database rename file '<old_path>/redo01.log' to '+DATA';
   ```
   
   -- Option B: Create fresh groups and drop old ones

   ```
   alter database add logfile thread 1 group 1 '+DATA' size 1G;
   alter database add logfile thread 1 group 2 '+DATA' size 1G;
   alter database add logfile thread 2 group 3 '+DATA' size 1G;
   alter database add logfile thread 2 group 4 '+DATA' size 1G;
   ```
   *Enable thread 2 after instance 2 creation; can be left defined for now*

3. **Create SPFILE in ASM and Set RAC-Specific Parameters**
     
   ```
   create spfile='+DATA/DBNAME/spfileDBNAME.ora' from pfile='/tmp/initDBNAME.ora';
   shutdown immediate;
   startup mount;
   ```
   -- Set RAC parameters
   
   ```
   alter system set cluster_database=true scope=spfile;
   alter system set instance_number=1 sid='DBNAME1' scope=spfile;
   alter system set thread=1 sid='DBNAME1' scope=spfile;
   alter system set undo_tablespace='UNDOTBS1' sid='DBNAME1' scope=both;
   ```
   
   -- Create UNDO for second instance
   
   ```
   create undo tablespace UNDOTBS2 datafile '+DATA' size 5G autoextend on;
   
   alter system set instance_number=2 sid='DBNAME2' scope=spfile;
   alter system set thread=2 sid='DBNAME2' scope=spfile;
   alter system set undo_tablespace='UNDOTBS2' sid='DBNAME2' scope=both;
   ```
   
   -- Set FRA, archiving, and other basics

   ```bash
   alter system set db_recovery_file_dest='+FRA' scope=both;
   alter system set db_recovery_file_dest_size='1T' scope=both;
   alter system set log_archive_dest_1='LOCATION=USE_DB_RECOVERY_FILE_DEST' scope=both;
   ```

---

### G. Open the Database
- **Normal Open (if no incomplete recovery):**  
  ```
  RMAN> sql 'alter database open';
  ```
- **RESETLOGS Open (if incomplete recovery used):**  
  ```
  RMAN> sql 'alter database open resetlogs';
  ```

---

### H. Register RAC Database and Instances in CRS on Target Cluster
1. **Add Database Resource**  
   ```
   srvctl add database -db DBNAME -oraclehome /u01/app/oracle/product/19c/dbhome_1 -dbname DBNAME -spfile +DATA/DBNAME/spfileDBNAME.ora -diskgroup DATA,FRA -role PRIMARY
   ```

2. **Add Instances Mapped to Nodes**  
   ```
   srvctl add instance -db DBNAME -instance DBNAME1 -node node1
   srvctl add instance -db DBNAME -instance DBNAME2 -node node2
   ```
   *Replace node and instance names as desired*

3. **Start and Verify**  
   ```
   srvctl start database -db DBNAME
   srvctl status database -db DBNAME
   ```

---


## Key Considerations and Tips
- Ensure source and target database versions and RU levels are compatible
- For cross-version restore, follow documented upgrade/downgrade paths after restore
- Keep CONTROLFILE AUTOBACKUP enabled; capture SPFILE in backup for smoother restore
- If moving ASM disk groups instead of restoring, mount existing disk groups on target and register database with srvctl (your plan uses RMAN restore)
- For ACFS use cases, ensure ASM proxy instance exists if ADVM/ACFS resources will be enabled later

---

## Summary
1. Take RMAN database+archivelog backup with controlfile autobackup on source cluster
2. Copy backups to target
3. Create minimal PFILE
4. Restore controlfile and mount
5. Catalog backups if paths differ
6. Restore and recover database into +DATA and +FRA
7. Create/configure redo threads and RAC instance parameters
8. Create SPFILE in ASM
9. Register database and instances with srvctl
10. Start RAC database on both nodes
