I'll help you create a comprehensive plan to backup the source RAC database and restore it in the target cluster with a new name and paths. Here's the step-by-step approach:

## **Phase 1: Backup from Source Cluster**

### **1. Backup Preparation on Source**
```sql
-- Connect to source database (any node)
sqlplus / as sysdba

-- Create pfile from spfile
create pfile='/tmp/initORCL.ora' from spfile;

-- Verify database configuration
show parameter db_name
show parameter db_unique_name
show parameter control_files
show parameter log_archive_dest
```

### **2. Perform RMAN Backup**
```bash
# Backup script on source cluster
rman target /

RUN {
    # Configure backup settings
    CONFIGURE CONTROLFILE AUTOBACKUP ON;
    CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '+FRA/%F';
    
    # Perform full database backup
    BACKUP DATABASE PLUS ARCHIVELOG;
    
    # Backup current controlfile to trace
    ALTER DATABASE BACKUP CONTROLFILE TO TRACE;
    
    # Backup spfile
    BACKUP SPFILE;
}
```

### **3. Copy Backup Files to Target**
```bash
# Identify backup locations
ASMCMD> ls +FRA/ORCL/BACKUPSET/

# Copy backup files to target cluster (using scp or shared storage)
# You can either:
# 1. Use shared NFS accessible by both clusters
# 2. Copy directly to target ASM using ASMCMD if shared filesystem exists
# 3. Use scp to copy to target nodes
```

## **Phase 2: Restore to Target Cluster**

### **1. Prepare Target ASM Environment**
```bash
# On target cluster, verify ASM diskgroups
asmcmd
ASMCMD> lsdg
```

### **2. Restore Control File with New Name**
```bash
# On target node1
rman auxiliary /

# In RMAN, restore controlfile with new name
RUN {
    SET NEWNAME FOR DATABASE TO '+DATA/TARGET';
    
    RESTORE CONTROLFILE FROM '<backup_location>/controlfile_backup';
    ALTER DATABASE MOUNT;
}
```

### **3. Restore Database with New Name**
```bash
# Create init file for new database on target
cat > $ORACLE_HOME/dbs/initTARGET1.ora << EOF
*.db_name='TARGET'
*.db_unique_name='TARGET'
*.instance_name='TARGET1'
*.instance_number=1
*.thread=1
*.undo_tablespace='UNDOTBS1'
*.db_create_file_dest='+DATA'
*.db_recovery_file_dest='+FRA'
*.db_recovery_file_dest_size=100G
*.cluster_database=true
*.remote_listener='LISTENER_TARGET'
*.compatible='19.0.0'
EOF
```

### **4. RMAN Restore Script**
```bash
rman target /

RUN {
    # Set new database name
    SET NEWNAME FOR DATABASE TO '+DATA/TARGET';
    
    # Restore database
    RESTORE DATABASE;
    
    # Switch all datafiles to new names
    SWITCH DATAFILE ALL;
    
    # Recover database
    RECOVER DATABASE;
}
```

### **5. Post-Restore Steps**
```sql
-- Open database with resetlogs
ALTER DATABASE OPEN RESETLOGS;

-- Recreate temp files
ALTER TABLESPACE TEMP ADD TEMPFILE SIZE 1G;

-- Recreate online redo logs in new location
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE ADD LOGFILE THREAD 1 GROUP 1 SIZE 200M;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE ADD LOGFILE THREAD 1 GROUP 2 SIZE 200M;
-- Repeat for all redo log groups

-- For RAC, add second thread
ALTER DATABASE ADD LOGFILE THREAD 2 GROUP 3 SIZE 200M;
ALTER DATABASE ADD LOGFILE THREAD 2 GROUP 4 SIZE 200M;

-- Enable thread 2 for second instance
ALTER DATABASE ENABLE PUBLIC THREAD 2;

-- Recreate spfile in ASM
CREATE SPFILE='+CRS/TARGET/spfileTARGET.ora' FROM PFILE;
```

## **Phase 3: Configure RAC on Target**

### **1. Create Second Instance**
```bash
# On target node2
cat > $ORACLE_HOME/dbs/initTARGET2.ora << EOF
*.db_name='TARGET'
*.db_unique_name='TARGET'
*.instance_name='TARGET2'
*.instance_number=2
*.thread=2
*.undo_tablespace='UNDOTBS2'
*.db_create_file_dest='+DATA'
*.db_recovery_file_dest='+FRA'
*.db_recovery_file_dest_size=100G
*.cluster_database=true
*.remote_listener='LISTENER_TARGET'
EOF

# Add second instance to cluster
srvctl add instance -d TARGET -i TARGET2 -n target2
```

### **2. Register Database with Clusterware**
```bash
# Register database with srvctl
srvctl add database -d TARGET -o $ORACLE_HOME \
  -p '+CRS/TARGET/spfileTARGET.ora' \
  -n TARGET \
  -r PRIMARY \
  -s OPEN \
  -a DATA,FRA

srvctl add service -d TARGET -s TARGET_SERVICE -r TARGET1,TARGET2

# Start database
srvctl start database -d TARGET
```

## **Phase 4: Post-Migration Verification**

### **1. Verify OMF Configuration**
```sql
-- Check OMF is working
SELECT name FROM v$datafile;
SELECT member FROM v$logfile;
SELECT name FROM v$controlfile;

-- Test OMF by creating new tablespace
CREATE TABLESPACE test_omf;
-- Verify file created in +DATA/TARGET
```

### **2. Update Global Database Name**
```sql
-- Update global name if needed
ALTER DATABASE RENAME GLOBAL_NAME TO TARGET.WORLD;
```

## **Alternative Approach: Using DUPLICATE DATABASE**

If you prefer a single-step approach:

```bash
rman target sys@source auxiliary sys@target

RUN {
    ALLOCATE AUXILIARY CHANNEL c1 DEVICE TYPE DISK;
    ALLOCATE AUXILIARY CHANNEL c2 DEVICE TYPE DISK;
    
    DUPLICATE DATABASE TO TARGET
    DB_FILE_NAME_CONVERT=('+DATA/ORCL','+DATA/TARGET')
    LOGFILE
      GROUP 1 SIZE 200M,
      GROUP 2 SIZE 200M,
      GROUP 3 SIZE 200M,
      GROUP 4 SIZE 200M;
}
```

## **Important Considerations:**

1. **Compatibility**: Ensure both clusters have same Oracle version
2. **ASM Compatibility**: Verify ASM compatibility settings
3. **Storage**: Ensure target ASM has sufficient space
4. **Network**: Configure tnsnames.ora and listeners on target
5. **Parameters**: Review and adjust all initialization parameters
6. **Password Files**: Create new password file for target database

This approach maintains OMF configuration while changing the database name and storage paths. The key is using `SET NEWNAME` in RMAN and proper ASM path mapping during restore.
