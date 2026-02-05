
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
3. Build a single parameter file on the target suitable for RAC 
4. Use RMAN Duplicate with backup-based 
5. Register and configure the RAC database resource with srvctl
6. Create instances on both nodes

---

## Detailed Steps

### A. On the Source Cluster: Create a Complete RMAN Backup

1. **Take Backup to a Filesystem or NFS Share**
   
   ```
   RMAN> run {
       allocate channel c1 type disk format '/tmp/backup/%U';
       backup database plus archivelog;
       backup current controlfile format '/backup/control_%U';
       backup spfile format '/tmp/backup/spfile_%U'
   }
   ```

### B. Transfer Backups to the Target Cluster
- Copy all backup pieces to a target directory (e.g., `/tmp/backup`)  
- One node access is sufficient for restore; ensure both nodes can access if needed

---

### C. Prepare the Target RAC Database Initialization Parameters
1. **Create Minimal PFILE on Target for NOMOUNT Startup**  
   File: `/tmp/initDBNAME.ora`  
   Example Contents:
   ```
   db_name=new_db_name
   db_unique_name=new_db_name
   cluster_database=false
   enable_pluggable_database=true
   compatible='19.0.0'
   instance_number=1
   ```
   *Adjust parameters to match your environment*

---


### D. Duplicate Database Backup into ASM

1. Start the instance using pfile
   
   ```bash
   sqlplus / as sysdba
   ```
   ```
   startup nomount pfile='<path_to_pfile>';
   ```

2. connect to the auxiliary instance in rman 
  ```
  rman auxiliary /
  ```
  
  - catalog the backup if backup_path differs
    
  ```
  catalog start with '<path_to_backup>';
  ```

**Example RMAN Run Block**  

```
run {
	SET NEWNAME FOR DATABASE TO NEW;
	DUPLICATE DATABASE TO '<new_db_name>'
		BACKUP LOCATION '/tmp/backup'
		NOFILENAMECHECK;
}
```
*Note: For point-in-time recovery, use `SET UNTIL {SCN|TIME|SEQUENCE}`*

3. create pfile from spfile to update the pfile

  ```
  create pfile from spfile ;
  ```

---
E. Create spfile in ASM and Modify the location of the spfile 

1. start the instance using pfile

   ```
   startup nomount pfile='<path_to_pfile>';
   ```
2. create PATAMETERFILE directory in ASM and create spfile in it

   ```
   mkdir +DATA/TARGET/PARAMETERFILE
   ```
   
   - create spfile in the new location in ASM
     
   ```
   create spfile='+DATA/TARGET/PARAMETERFILE/spfileTEST.ora' from pfile='<path_to_pfile>' ;
   ```
3. add the spfile location parameter in the pfile

   ```
   vi <path_to_pfile>
   ```
   
   - add the following parameter
     
   ```
   spfile='+DATA/TARGET/PARAMETERFILE/spfileTARGET.ora'
   ```

--- 
### F. Prepare Instance/RAC Settings

1. **Set RAC-Specific Parameters**
     
   ```
   alter system set cluster_database=true scope=spfile;
   alter system set instance_number=1 sid='DBNAME1' scope=spfile;
   alter system set thread=1 sid='DBNAME1' scope=spfile;
   alter system set undo_tablespace='UNDOTBS1' sid='DBNAME1' scope=both;
   ```
      
   ```   
   alter system set instance_number=2 sid='DBNAME2' scope=spfile;
   alter system set thread=2 sid='DBNAME2' scope=spfile;
   alter system set undo_tablespace='UNDOTBS2' sid='DBNAME2' scope=both;
   ```
   
   -- Set FRA, archiving, and other basics

   ```bash
   alter system set db_recovery_file_dest='+FRA' scope=both;
   alter system set db_recovery_file_dest_size='<size>' scope=both;
   alter system set log_archive_dest_1='LOCATION=USE_DB_RECOVERY_FILE_DEST' scope=both;
   ```

---

### G. Register RAC Database and Instances in CRS on Target Cluster
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
