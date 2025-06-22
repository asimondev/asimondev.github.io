+++
title = 'Selecting AWR SQL Statistics In CDB'
date = 2025-06-22T16:06:32+02:00
draft = true
summary = """
How to fetch SQL statement statistics from AWR in a multitenant database.
"""
tags = ["awr", "tuning"]
+++

In a multitenant enviroment the AWR data could be collected both on CDB and PDB level. In this blog I would like to clarify this topic.

## Creating Test Environment

The tests are executed on Oracle database 19.26. At first I would like to create some PDBs. Each PDBs will contain a specific database user with 2 tables. The AWR data will be collected both global and on PDB level. 

### Changes in CDB$ROOT

```
-- Enable AWR in all PDBs
alter system set awr_pdb_autoflush_enabled=true;

-- Set snapshot time offset in seconds
alter system set awr_snapshot_time_offset=300;
```

### Creating Test PDBs

I am going to create 3 PDBs. Additionally, the AWR snapshots will be created in CDB$ROOT before and after creating PDBs. Every PDB contains unique database user. This user owns 2 tables (*CREATE TABLE AS SELECT * FROM DBA_OBJECTS). One table is MY_OBJECTS and another table has got unique name.

| Name | Database User | Tables |
|------|---------------|--------|
| CDBA2_PDB1 | ANDREJ1 | MY_OBJECTS, MY_OBJECTS1 |
| CDBA2_PDB2 | ANDREJ2 | MY_OBJECTS, MY_OBJECTS2 |
| CDBA2_PDB3 | ANDREJ3 | MY_OBJECTS, MY_OBJECTS3 |

Here is the overview of the PDBs and created snapshots:

```
SQL> select con_id, snap_id, begin_interval_time, end_interval_time
  2  from cdb_hist_snapshot 
  3  where (con_id = 0 and snap_id > 32) or con_id > 0
  4  order by con_id, snap_id;

   CON_ID    SNAP_ID BEGIN_INTERVAL_TIME                END_INTERVAL_TIME                  
_________ __________ __________________________________ __________________________________ 
        0         33 22-JUN-25 05.07.32.133000000 PM    22-JUN-25 05.09.55.138000000 PM    
        0         34 22-JUN-25 05.09.55.138000000 PM    22-JUN-25 05.10.56.816000000 PM    
        0         35 22-JUN-25 05.10.56.816000000 PM    22-JUN-25 05.11.47.933000000 PM    
        0         36 22-JUN-25 05.11.47.933000000 PM    22-JUN-25 05.12.56.008000000 PM    
        0         37 22-JUN-25 05.12.56.008000000 PM    22-JUN-25 05.12.56.350000000 PM    
        4          1 22-JUN-25 03.12.29.000000000 PM    22-JUN-25 05.10.11.029000000 PM    
        4          2 22-JUN-25 05.10.11.029000000 PM    22-JUN-25 05.10.55.874000000 PM    
        6          1 22-JUN-25 03.12.29.000000000 PM    22-JUN-25 05.11.07.029000000 PM    
        6          2 22-JUN-25 05.11.07.029000000 PM    22-JUN-25 05.11.46.985000000 PM    
        7          1 22-JUN-25 03.12.29.000000000 PM    22-JUN-25 05.11.54.359000000 PM    
        7          2 22-JUN-25 05.11.54.359000000 PM    22-JUN-25 05.12.54.971000000 PM    

11 rows selected. 

SQL> 
SQL> show pdbs

   CON_ID CON_NAME      OPEN MODE     RESTRICTED    
_________ _____________ _____________ _____________ 
        2 PDB$SEED      READ ONLY     NO            
        4 CDBA2_PDB1    READ WRITE    NO            
        6 CDBA2_PDB2    READ WRITE    NO            
        7 CDBA2_PDB3    READ WRITE    NO            

```

## Available AWR Views

A non-CDB database contains only *DBA_HIST* views. The Oracle multitenant database provides *DBA_HIST*, *AWR_ROOT*, *AWR_PDB* and *CDB_HIST* views. 

See: [Oracle Instance Tuning](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/gathering-database-statistics.html#GUID-D64AEB01-18FF-47EF-BB5C-A0611117D180)

| Current Container | DBA_HIST | AWR_ROOT | AWR_PDB | CDB_HIST |
|-------------------|----------|----------|---------|----------|
| CDB$ROOT | CDB root AWR data | CDB root AWR data | CDB root AWR data | Union of all AWR PDB data |
| PDB | Subset of CDB root AWR data | Subset of CDB root AWR data | PDB AWR data | PDB AWR data |

In fact the *DBA_HIST* and *AWR_ROOT* views contains the same data. We will now compare *AWR_ROOT*, *AWR_PDB* and *CDB_HIST* views.

