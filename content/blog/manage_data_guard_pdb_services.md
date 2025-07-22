+++
title = 'Managing PDBs in Data Guard Broker / RAC Environment'
date = 2025-07-22T20:46:09+02:00
draft = true
summary = """
Using Data Guard Broker in RAC environment is not difficult. But the Multitenant Architecture adds more complexity. This blog describes how to manage RAC services with PDBs in Data Guard environment.
"""
tags = ["rac", "standby", "data guard", "pdb"]
+++

The example commands use the following configuration.

There are 3 RAC clusters consisting of the nodes *bol7rac2a*/*bol7rac2b*, *rkol7rac2a*/*rkol7rac2b* and *aol7rac2a*/*aol7rac2b*. The primary RAC database *cdba02* runs on the servers *bol7rac1* and *bol7rac2* and uses ASM. The database unique name is *cdba02_dc1*. The database domain is *world*. The standby databases have unique names *cdba02_dc2* and *cdba02_dc3*.

In my previous blog [How to Add a Standby RAC Database?]({{< relref "create_standby_rac_db.md" >}}) I have already provided the basic steps for creating a new RAC standby database. Two standby databases were created using these steps. 

## Current Configuration

Here is the output of the current Data Guard Broker Configuration:

```
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxAvailability
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 
    cdba02_dc3 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 59 seconds ago)
```

Usually a new standby database would be created by SRVCTL CONFIG DATABASE with *PHYSICAL_STANDBY* database role and *mount* start option. The current primary database would have a *PRIMARY* role and *open* start option.

### Primary Database CDBA02_DC1
```
oracle@bol7rac2a> srvctl config database -db cdba02_dc1
Database unique name: cdba02_dc1
Database name: cdba02
...
Start options: open
...
Database role: PRIMARY
...
```

### Standby Database CDBA02_DC2
```
oracle@rkol7rac2a> srvctl config database -db cdba02_dc2
Database unique name: cdba02_dc2
Database name: cdba02
...
Start options: mount
...
Database role: PHYSICAL_STANDBY
...
```

### Standby Database CDBA02_DC3
```
oracle@aol7rac2a> srvctl config database -db cdba02_dc3
Database unique name: cdba02_dc3
Database name: cdba02
...
Start options: mount
...
Database role: PHYSICAL_STANDBY
...
```

## Creating RAC Services

This database contains one PDB:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      READ ONLY  NO
	 3 CDBA02_PDB1			  READ WRITE NO
```

We can use the following script *add_services.sh* to create 3 services with different roles on the primary database:
```
oracle@bol7rac2a> cat add_services.sh 
#!/bin/bash -x

DB_NAME=cdba02
DB_UNIQUE_NAME=${DB_NAME}_dc1

srvctl add service -db $DB_UNIQUE_NAME -service prod.world -role PRIMARY -pdb cdba02_pdb1 -preferred cdba021,cdba022
srvctl add service -db $DB_UNIQUE_NAME -service read_only.world -role "PHYSICAL_STANDBY" -pdb cdba02_pdb1 -preferred cdba021,cdba022
srvctl add service -db $DB_UNIQUE_NAME -service always.world -role "PRIMARY,PHYSICAL_STANDBY" -pdb cdba02_pdb1 -preferred cdba021,cdba022
oracle@bol7rac2a> 

oracle@bol7rac2a> ./add_services.sh 
+ DB_NAME=cdba02
+ DB_UNIQUE_NAME=cdba02_dc1
+ srvctl add service -db cdba02_dc1 -service prod.world -role PRIMARY -pdb cdba02_pdb1 -preferred cdba021,cdba022
+ srvctl add service -db cdba02_dc1 -service read_only.world -role PHYSICAL_STANDBY -pdb cdba02_pdb1 -preferred cdba021,cdba022
+ srvctl add service -db cdba02_dc1 -service always.world -role PRIMARY,PHYSICAL_STANDBY -pdb cdba02_pdb1 -preferred cdba021,cdba022
oracle@bol7rac2a> 
```

We can copy this script to another clusters, modify the DB_UNIQUE_NAME variable and execute it.

Now we would like to restart all 3 database and check the services to get a *clean* start.

### Primary Database CDBA02_DC1
```
oracle@bol7rac2a> srvctl stop database -db cdba02_dc1
oracle@bol7rac2a> srvctl start database -db cdba02_dc1
oracle@bol7rac2a> srvctl status service -db cdba02_dc1 
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is running on instance(s) cdba021,cdba022
Service read_only.world is not running.
```
The service *read_only.world* is the only service with a physical standby role. It should not run on the primary database.

Status of PDB in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      READ ONLY  NO
	 3 CDBA02_PDB1			  READ WRITE NO
```

### Standby Database CDBA02_DC2
```
oracle@rkol7rac2a> srvctl stop database -db cdba02_dc2
oracle@rkol7rac2a> srvctl start database -db cdba02_dc2 -startoption mount

oracle@rkol7rac2a> srvctl status service -db cdba02_dc2
Service always.world is not running.
Service prod.world is not running.
Service read_only.world is not running.
oracle@rkol7rac2a> 
```

Database status in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      MOUNTED
	 3 CDBA02_PDB1			  MOUNTED

SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
MOUNTED 	     PHYSICAL STANDBY

```

No RAC service is running because the database is not opened.

### Standby Database CDBA02_DC3
```
oracle@aol7rac2a> srvctl stop database -db cdba02_dc3
oracle@aol7rac2a> srvctl start database -db cdba02_dc3 -startoption mount

oracle@aol7rac2a> srvctl status service -db cdba02_dc3
Service always.world is not running.
Service prod.world is not running.
Service read_only.world is not running.
```

Database status in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      MOUNTED
	 3 CDBA02_PDB1			  MOUNTED

SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
MOUNTED 	     PHYSICAL STANDBY
```

No RAC service is running because the database is not opened.

## Removing PDB States in Primary Database

Per default a PDB would start in a mount mode. A CDB contains a few PDBs. Usually you would open the PDBs and save their state. The corresponding PDBs would be opened during the next start of the CDB. 

You can save the state only to the primary database (READ WRITE). But you have a primary database and standby databases. You can't save the state of the standby (READ ONLY) database. 

The solution is not to save the states of PDBs at all. You should use the RAC services configured for each database to achieve the same goal. 

That's why the first step should be to delete the possible saved states on the primary database:
```
SQL> alter pluggable database all discard state instances=all;

Pluggable database altered.

SQL> select * from dba_pdb_saved_states;

no rows selected

SQL> 
```

## First Switchover

We have configured the RAC services on all 3 RAC databases. The primary database is open and both standby databases are in the mount mode. Let's try to switchover to the first standby database.
```
oracle@bol7rac2a> dgmgrl 
...
DGMGRL> connect sys/oracle@bscan2/cdba02_dc1.world
Connected to "cdba02_dc1"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxAvailability
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 
    cdba02_dc3 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 28 seconds ago)

DGMGRL> switchover to cdba02_dc2;
Performing switchover NOW, please wait...
Operation requires a connection to database "cdba02_dc2"
Connecting ...
Connected to "cdba02_dc2"
Connected as SYSDBA.
New primary database "cdba02_dc2" is opening...
Oracle Clusterware is restarting database "cdba02_dc1" ...
Connected to "cdba02_dc1"
Switchover succeeded, new primary is "cdba02_dc2"
```

### New Primary Database CDBA02_DC2

```
oracle@rkol7rac2a> srvctl config database -db cdba02_dc2
Database unique name: cdba02_dc2
Database name: cdba02
...
Start options: open
...
Database role: PRIMARY
...
oracle@rkol7rac2a> srvctl status service -db cdba02_dc2
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is running on instance(s) cdba021,cdba022
Service read_only.world is not running.
oracle@rkol7rac2a> 
```

The corresponding services are started. The start option changed to *open* and the database role to *PRIMARY*.

The Data Guard Broker works together with Oracle RAC. During switchover the new primary get some cluster properties updated. The old primary database would be started with the same start option as the previous standby database.

### Old Primary / New Standby Database CDBA02_DC1

```
oracle@bol7rac2a> srvctl config database -db cdba02_dc1
Database unique name: cdba02_dc1
Database name: cdba02
...
Start options: mount
...
Database role: PHYSICAL_STANDBY
...
oracle@bol7rac2a> srvctl status service -db cdba02_dc1 
Service always.world is not running.
Service prod.world is not running.
Service read_only.world is not running.
oracle@bol7rac2a> 
```

Database status in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      MOUNTED
	 3 CDBA02_PDB1			  MOUNTED
SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
MOUNTED 	     PHYSICAL STANDBY
```

After switchover the database is mounted but not opened. The PDB is not open. The RAC services are not running anymore.

If you only use standby database in the mount status, you don't have to change anything. This configuration works now.

## Opened Standby Databases

In some Data Guard Broker Configurations you want to use opened standby databases (Active Data Guard) for different reasons. You can open the standby database in READ ONLY mode. This would allow managed recovery process to apply redo logs. At the same time this would allow the RAC services to start and open the corresponding PDBs. 

We can reset the start option for the standby databases and restart them.

### Standby Database CDBA02_DC1

```
oracle@bol7rac2a> srvctl modify database -db cdba02_dc1 -startoption open
oracle@bol7rac2a> srvctl stop database -db cdba02_dc1
oracle@bol7rac2a> srvctl start database -db cdba02_dc1

oracle@bol7rac2a> srvctl status service -db cdba02_dc1 
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is not running.
Service read_only.world is not running.
```

Database status in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      READ ONLY  NO
	 3 CDBA02_PDB1			  READ ONLY  NO
SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY
```

The database was opened because of the *open* start option. The RAC services were started because the database role is *physical standby*. Starting of the RAC services also opened the PDB.

### Standby Database CDBA02_DC3

This standby database looks similar:

```
oracle@aol7rac2a> srvctl modify database -db cdba02_dc3 -startoption open
oracle@aol7rac2a> srvctl stop database -db cdba02_dc3
oracle@aol7rac2a> srvctl start database -db cdba02_dc3

oracle@aol7rac2a> srvctl status service -db cdba02_dc3
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is not running.
Service read_only.world is not running.
```

Database status in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      READ ONLY  NO
	 3 CDBA02_PDB1			  READ ONLY  NO
SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY
```

## Second Switchover

Let's run the switchover to the second standby database:

```
oracle@rkol7rac2a> dgmgrl  
...
DGMGRL> connect sys/oracle@rscan2/cdba02_dc2.world
Connected to "cdba02_dc2"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxAvailability
  Members:
  cdba02_dc2 - Primary database
    cdba02_dc1 - Physical standby database 
    cdba02_dc3 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 30 seconds ago)

DGMGRL> switchover to cdba02_dc3;
Performing switchover NOW, please wait...
Operation requires a connection to database "cdba02_dc3"
Connecting ...
Connected to "cdba02_dc3"
Connected as SYSDBA.
New primary database "cdba02_dc3" is opening...
Oracle Clusterware is restarting database "cdba02_dc2" ...
Connected to "cdba02_dc2"
Switchover succeeded, new primary is "cdba02_dc3"
```

Let's check the old primary database:
```
oracle@rkol7rac2a> srvctl config database -db cdba02_dc2
Database unique name: cdba02_dc2
Database name: cdba02
...
Start options: read only
...
Database role: PHYSICAL_STANDBY
...
```

Database status in SQL*Plus:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      READ ONLY  NO
	 3 CDBA02_PDB1			  READ ONLY  NO
SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY
```

Data Guard Broker worked again with Oracle Clusterware. The new start option is set to *read only* and the database role to *physical standby*. The RAC services with the standby role were started and the PDB was opened.

## Conclusion

- Do not use saved states in RAC CDBs with Data Guard Broker.
- Use RAC services to open the PDBs
- Use Clusterware RAC Database start option to mount or open a starting RAC database.
- Data Guard Broker uses Oracle Clusterware after switchover to start the new standby database with same start option as the previous primary database.
