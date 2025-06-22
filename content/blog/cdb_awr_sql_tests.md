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
  3  where (con_id = 0 and snap_id > 46) or con_id > 0
  4  order by con_id, snap_id;

   CON_ID    SNAP_ID BEGIN_INTERVAL_TIME                END_INTERVAL_TIME                  
_________ __________ __________________________________ __________________________________ 
        0         47 22-JUN-25 06.30.30.798000000 PM    22-JUN-25 06.30.45.998000000 PM    
        0         48 22-JUN-25 06.30.45.998000000 PM    22-JUN-25 06.31.29.375000000 PM    
        0         49 22-JUN-25 06.31.29.375000000 PM    22-JUN-25 06.31.29.872000000 PM    
        0         50 22-JUN-25 06.31.29.872000000 PM    22-JUN-25 06.33.05.949000000 PM    
        0         51 22-JUN-25 06.33.05.949000000 PM    22-JUN-25 06.33.08.692000000 PM    
        0         52 22-JUN-25 06.33.08.692000000 PM    22-JUN-25 06.33.56.907000000 PM    
        0         53 22-JUN-25 06.33.56.907000000 PM    22-JUN-25 06.33.57.373000000 PM    
        0         54 22-JUN-25 06.33.57.373000000 PM    22-JUN-25 06.33.57.662000000 PM    
        6          1 22-JUN-25 03.12.29.000000000 PM    22-JUN-25 06.33.15.176000000 PM    
        8          1 22-JUN-25 03.12.29.000000000 PM    22-JUN-25 06.30.52.980000000 PM    
        9          1 22-JUN-25 03.12.29.000000000 PM    22-JUN-25 06.31.46.889000000 PM    

11 rows selected. 

SQL> show pdbs

   CON_ID CON_NAME      OPEN MODE     RESTRICTED    
_________ _____________ _____________ _____________ 
        2 PDB$SEED      READ ONLY     NO            
        6 CDBA2_PDB3    READ WRITE    NO            
        8 CDBA2_PDB1    READ WRITE    NO            
        9 CDBA2_PDB2    READ WRITE    NO            
   

SQL> select distinct con_id, dbid, con_dbid, pdb_name from dba_hist_pdb_instance where con_id is not null order by con_id
  2* ;

   CON_ID          DBID      CON_DBID PDB_NAME      
_________ _____________ _____________ _____________ 
        1    2451738424    2451738424 CDB$ROOT      
        2    2451738424    3036536457 PDB$SEED      
        6    2451738424     586577114 CDBA2_PDB3    
        8    2451738424    4078008593 CDBA2_PDB1    
        9    2451738424    3400071985 CDBA2_PDB2    
```

## Available AWR Views

A non-CDB database contains only *DBA_HIST* views. The Oracle multitenant database provides *DBA_HIST*, *AWR_ROOT*, *AWR_PDB* and *CDB_HIST* views. 

See: [Oracle Instance Tuning](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/gathering-database-statistics.html#GUID-D64AEB01-18FF-47EF-BB5C-A0611117D180)

| Current Container | DBA_HIST | AWR_ROOT | AWR_PDB | CDB_HIST |
|-------------------|----------|----------|---------|----------|
| CDB$ROOT | CDB root AWR data | CDB root AWR data | CDB root AWR data | Union of all AWR PDB data |
| PDB | Subset of CDB root AWR data | Subset of CDB root AWR data | PDB AWR data | PDB AWR data |

In fact the *DBA_HIST* and *AWR_ROOT* views contains the same data. We will now compare *AWR_ROOT*, *AWR_PDB* and *CDB_HIST* views.

### CDB_HIST Views

#### CDB$ROOT Container

In all 3 PDB I have executed as local user *ANDREJ1*, *ANDREJ2* and *ANDREJ3* the following statement:\
`update my_objects3 set char2='ANDREJ'`

Let's try to find this statement from the *CDB$ROOT* container.

```
SQL> select a.sql_id, b.snap_id, a.dbid, a.con_id, a.con_dbid, a.sql_text sqltext, c.username,
  2  b.parsing_schema_name pschema, b.parsing_user_id puser_id, executions_delta execs
  3  from cdb_hist_sqltext a,
  4  cdb_hist_sqlstat b,
  5  cdb_users c
  6  where a.dbid = b.dbid and a.con_id = b.con_id and a.con_dbid = b.con_dbid and a.sql_id = b.sql_id and
  7  b.con_id = c.con_id and b.parsing_user_id = c.user_id and
  8  a.sql_text like '%update%char2%'
  9  order by sql_id, con_id, snap_id
 10  /

SQL_ID		 SNAP_ID       DBID	CON_ID	 CON_DBID SQLTEXT				   USERNAME   PSCHEMA	 PUSER_ID      EXECS
------------- ---------- ---------- ---------- ---------- ---------------------------------------- ---------- ---------- -------- ----------
9mtn2g4w58mvk	       2  586577114	     6	586577114 update my_objects3 set char2='ANDREJ'    ANDREJ3    ANDREJ3	      110	   1
9mtn2g4w58mvk	      52 2451738424	     6	586577114 update my_objects3 set char2='ANDREJ'    ANDREJ3    ANDREJ3	      110	   1
ap1239wc3j0rx	       2 4078008593	     8 4078008593 update my_objects1 set char2='ANDREJ'    ANDREJ1    ANDREJ1	      110	   1
ap1239wc3j0rx	      48 2451738424	     8 4078008593 update my_objects1 set char2='ANDREJ'    ANDREJ1    ANDREJ1	      110	   1
gn1w3mg3rqvzm	       2 3400071985	     9 3400071985 update my_objects2 set char2='ANDREJ'    ANDREJ2    ANDREJ2	      110	   1
gn1w3mg3rqvzm	      50 2451738424	     9 3400071985 update my_objects2 set char2='ANDREJ'    ANDREJ2    ANDREJ2	      110	   1

6 rows selected.
```

Apparently I could find this SQL statement in all 3 PDBs. The statement was found in 2 snapshots per PDB:
- In global AWR data (for instance, SNAP_ID *52* for PDB with CON_ID *6*)
- In local AWR data (for instance, SNAP_ID *2* for PDB with CON_ID *6*)

The *CDB_HIST_SQLSTAT* contains the column *PARSING_USER_ID*. I could also join this column to *CDB_USERS* and get the corresponding username for each PDB.

#### PDB Container

Let's try to find the same statement from *CDBA2_PDB1* container.

```
SQL> alter session set container=cdba2_pdb1;

Session altered.

SQL> show con_name

CON_NAME
------------------------------
CDBA2_PDB1

SQL> set echo on
SQL> spool check_pdb_cdb_hist.log
SQL> 
SQL> set linesi 255 trimsp on pagesi 100
SQL> col username for a10
SQL> col pschema for a10
SQL> col puser_id for 999
SQL> col sqltext for a40
SQL> 
SQL> select a.sql_id, b.snap_id, a.dbid, a.con_id, a.con_dbid, a.sql_text sqltext, c.username,
  2  b.parsing_schema_name pschema, b.parsing_user_id puser_id, executions_delta execs
  3  from cdb_hist_sqltext a,
  4  cdb_hist_sqlstat b,
  5  cdb_users c
  6  where a.dbid = b.dbid and a.con_id = b.con_id and a.con_dbid = b.con_dbid and a.sql_id = b.sql_id and
  7  b.con_id = c.con_id and b.parsing_user_id = c.user_id and
  8  a.sql_text like '%update%char2%'
  9  order by sql_id, con_id, snap_id
 10  /

SQL_ID		 SNAP_ID       DBID	CON_ID	 CON_DBID SQLTEXT				   USERNAME   PSCHEMA	 PUSER_ID      EXECS
------------- ---------- ---------- ---------- ---------- ---------------------------------------- ---------- ---------- -------- ----------
ap1239wc3j0rx	       2 4078008593	     8 4078008593 update my_objects1 set char2='ANDREJ'    ANDREJ1    ANDREJ1	      110	   1

SQL> 
```

So the same SELECT from the PDB can only access the local snapshot (SNAP_ID *2*). We can't access other statements from another PDBs.

