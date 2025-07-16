+++
title = 'How To Add RAC Services'
date = 2025-07-16T20:26:10+02:00
draft = false
summary = """
If you use RAC and Data Guard together, you can create RAC database services for different database roles (primary or standby). This blog shows how to do this.
"""
tags = ["rac", "standby", "data guard"]
+++

The example commands use the following configuration.

There are 2 RAC clusters consisting of the nodes *bol7rac2a*/*bol7rac2b* and *rkol7rac2a*/*rkol7rac2b*. The primary RAC database *cdba02* runs on the servers *bol7rac1* and *bol7rac2* and uses ASM. The database unique name is *cdba02_dc1*. The database domain is *world*. The standby database uses the database unique name *cdba02_dc2*. No services existed before. Here is output of Data Guard configuration:
```
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 5 seconds ago)
```

There is one PDB in this database:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 CDBA02_PDB1			  READ WRITE NO
SQL> 
```

## Creating Services on the Primary Database

I would like to add 3 services:
- *prod.world* for production applications using a primary database only
- *read_only.world* for read only applications using a standby database only
- *always.world* service running on both primary and standby databases.

All services should run on both cluster nodes.
```
srvctl add service -db cdba02_dc1 -service prod.world -role PRIMARY -pdb cdba02_pdb1 -preferred cdba021,cdba022
srvctl add service -db cdba02_dc1 -service read_only.world -role PHYSICAL_STANDBY -pdb cdba02_pdb1 -preferred cdba021,cdba022
srvctl add service -db cdba02_dc1 -service always.world -role PRIMARY,PHYSICAL_STANDBY -pdb cdba02_pdb1 -preferred cdba021,cdba022
```

You can check service configuration with SRVCTL CONFIG SERVICE command. 

Let's check, if the created services are running:
```
oracle@bol7rac2a> srvctl status service -db cdba02_dc1
Service always.world is not running.
Service prod.world is not running.
Service read_only.world is not running.
```

## Starting Services on the Primary Database

Let's check, if the created services are running:
```
oracle@bol7rac2a> srvctl status service -db cdba02_dc1
Service always.world is not running.
Service prod.world is not running.
Service read_only.world is not running.
```

We can start services using SRVCTL START SERVICE command:
```
srvctl start service -db cdba02_dc1 -service prod.world
srvctl start service -db cdba02_dc1 -service read_only.world
srvctl start service -db cdba02_dc1 -service always.world
```

And here is the status of the services:
```
oracle@bol7rac2a> srvctl status service -db cdba02_dc1
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is running on instance(s) cdba021,cdba022
Service read_only.world is running on instance(s) cdba021,cdba022
```

All services are running. But the service *read_only.world* should only run on the primary database! Where is the error? There is no error. We have started services manually. The *role* option only applied during the automatic start of the database. 

We can restart the database and check the services:
```
oracle@bol7rac2a> srvctl stop database -db cdba02_dc1

oracle@bol7rac2a> srvctl start database -db cdba02_dc1

oracle@bol7rac2a> srvctl status service -db cdba02_dc1
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is running on instance(s) cdba021,cdba022
Service read_only.world is not running.
oracle@bol7rac2a> 
```

All services are now running as expected.

## Creating Services on the Standby Database

Now we have to create the same services on the standby database. Don't forget to use the correct database unique name!

Let's check the database status in SQL*Plus:
```
SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
MOUNTED 	     PHYSICAL STANDBY
```

We can add services now:
```
srvctl add service -db cdba02_dc2 -service prod.world -role PRIMARY -pdb cdba02_pdb1 -preferred cdba021,cdba022
srvctl add service -db cdba02_dc2 -service read_only.world -role PHYSICAL_STANDBY -pdb cdba02_pdb1 -preferred cdba021,cdba022
srvctl add service -db cdba02_dc2 -service always.world -role PRIMARY,PHYSICAL_STANDBY -pdb cdba02_pdb1 -preferred cdba021,cdba022
```

## Starting Services on the Standby Database

Let's check, if the created services are running:
```
oracle@rkol7rac2a> srvctl status service -db cdba02_dc2
Service always.world is not running.
Service prod.world is not running.
Service read_only.world is not running.
```

We can start services using SRVCTL START SERVICE command:
```
 srvctl start service -db cdba02_dc2 -service prod.world
PRCD-1133 : failed to start services prod.world for database cdba02_dc2
PRCR-1095 : Failed to start resources using filter ((NAME == ora.cdba02_dc2.prod.world.svc) AND (TYPE == ora.service.type))
CRS-2800: Cannot start resource 'ora.cdba02_dc2.db' as it is already in the INTERMEDIATE state on server 'rkol7rac2b'
CRS-2632: There are no more servers to try to place resource 'ora.cdba02_dc2.prod.world.svc' on that would satisfy its placement policy
CRS-2800: Cannot start resource 'ora.cdba02_dc2.db' as it is already in the INTERMEDIATE state on server 'rkol7rac2a'
+ srvctl start service -db cdba02_dc2 -service read_only.world
PRCD-1133 : failed to start services read_only.world for database cdba02_dc2
PRCR-1095 : Failed to start resources using filter ((NAME == ora.cdba02_dc2.read_only.world.svc) AND (TYPE == ora.service.type))
CRS-2800: Cannot start resource 'ora.cdba02_dc2.db' as it is already in the INTERMEDIATE state on server 'rkol7rac2b'
CRS-2632: There are no more servers to try to place resource 'ora.cdba02_dc2.read_only.world.svc' on that would satisfy its placement policy
CRS-2800: Cannot start resource 'ora.cdba02_dc2.db' as it is already in the INTERMEDIATE state on server 'rkol7rac2a'
+ srvctl start service -db cdba02_dc2 -service always.world
PRCD-1133 : failed to start services always.world for database cdba02_dc2
PRCR-1095 : Failed to start resources using filter ((NAME == ora.cdba02_dc2.always.world.svc) AND (TYPE == ora.service.type))
CRS-2800: Cannot start resource 'ora.cdba02_dc2.db' as it is already in the INTERMEDIATE state on server 'rkol7rac2b'
CRS-2632: There are no more servers to try to place resource 'ora.cdba02_dc2.always.world.svc' on that would satisfy its placement policy
CRS-2800: Cannot start resource 'ora.cdba02_dc2.db' as it is already in the INTERMEDIATE state on server 'rkol7rac2a'
oracle@rkol7rac2a> 

```

Why do we get these errors? The standby database is mounted only. The database must be opened to start the services. This is a standby database, so we can open it *READ ONLY* using SQL*Plus:
```
SQL> alter database open read only;
alter database open read only
*
ERROR at line 1:
ORA-10456: cannot open standby database; media recovery session may be in
progress
```

We can use broker command in DGMGRL to stop the managed recovery on the standby database:
```
dgmgrl /
...
DGMGRL> show configuration

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 48 seconds ago)

DGMGRL> edit database cdba02_dc2 set state=apply-off;
Succeeded.
```

Again in SQL*Plus session on the standby database we can open this database as read only:
```
SQL> alter database open read only;

Database altered.
```

In DGMGRL session we should re-start managed recovery on this standby database:
```
DGMGRL> edit database cdba02_dc2 set state=apply-on;
Succeeded.
```

Let's check the status of the standby database and PDB:
```
SQL> select open_mode, database_role from v$database;

OPEN_MODE	     DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY

SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			  READ ONLY  NO
	 3 CDBA02_PDB1			  READ ONLY  NO
```

Let's check the status of the services:
```
oracle@rkol7rac2a> srvctl status service -db cdba02_dc2
Service always.world is running on instance(s) cdba021
Service prod.world is running on instance(s) cdba021
Service read_only.world is running on instance(s) cdba021
oracle@rkol7rac2a> 
```

We didn't start any services. But we have opened the database for read only. This triggered starting the services. All services are running. But the service *prod.world* should only run on the primary database!

We can restart the database and check the services. This time we would use the *startoption* "read only" instead of the usual "mount" option.

```
oracle@rkol7rac2a> srvctl stop database -db cdba02_dc2

oracle@rkol7rac2a> srvctl start database -db cdba02_dc2 -startoption "read only"

oracle@rkol7rac2a> srvctl status service -db cdba02_dc2
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is not running.
Service read_only.world is running on instance(s) cdba021,cdba022
oracle@rkol7rac2a> 
```

With that, all services are now operating properly.

## Data Guard Switchover

Let't try the Data Guard broker *switchover* command:
```
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 54 seconds ago)

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
DGMGRL> 
```

### Current Standby Database CDBA02_DC1

```
oracle@bol7rac2a> srvctl status service -db cdba02_dc1
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is not running.
Service read_only.world is running on instance(s) cdba021,cdba022
oracle@bol7rac2a> 
```

### Current Primary Database CDBA02_DC2

```
oracle@rkol7rac2a> srvctl status service -db cdba02_dc2
Service always.world is running on instance(s) cdba021,cdba022
Service prod.world is running on instance(s) cdba021,cdba022
Service read_only.world is not running.
oracle@rkol7rac2a> 
```

So even after switchover all services are running correctly.
