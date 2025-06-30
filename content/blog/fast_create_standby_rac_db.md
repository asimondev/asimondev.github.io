+++
title = 'How To Add a Large Standby Databse Fast'
date = 2025-06-30T18:45:35+02:00
draft = true
summary = """
There are some specific ways to work with a large database.  This document describes creating a new standby RAC database using RMAN as fast as possible. The new standby database will be added to the Data Guard Broker configuration.
"""
tags = ["rac", "standby", "rman", "data guard"]
+++

The example commands use the following configuration.

There are 2 RAC clusters consisting of the nodes *bol7rac2a*/*bol7rac2b* and *rkol7rac2a*/*rkol7rac2b*. The OS user "grid" was used to install the Grid Infrastructure. There are 2 ASM disk groups DATA and FRA on each cluster.

The RAC database was installed using the OS user "oracle". The primary RAC database *cdba02* runs on the servers *bol7rac1* and *bol7rac2* and uses ASM. The database unique name is *cdba02_dc1*. The database domain is *world*.

In my previous blog [How to Add a Standby RAC Database?]({{< relref "create_standby_rac_db.md" >}}) I have already provided the basis steps. The new steps should accelerate creating a new large standby database.

Specific topics compared to the previous blog:
> [!NOTE]
> - [Using a separate network for creating a new standby database](#using-second-network)


```
SQL> show parameter db_name

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_name 			     string	 cdba02
SQL> show parameter db_unique_name

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_unique_name			     string	 cdba02_dc1
SQL> show parameter domain

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_domain			     string	 world
```

The new standby database should run on the servers *rkol7rac1* and *rkol7rac2*. The database unique name will be *cdba02_dc2*.

## Preparing the Primary Database

### Set Up Primary Database

You have to enable **ARCHIVELOG** mode on the primary database. To avoid possible issues during standby recovery you have to set **Force Logging** as well. It's recommended to use **Flashback Database** especially on the standby database. 

```
SQL> select log_mode, force_logging from v$database;

LOG_MODE     FORCE_LOGGING
------------ ---------------------------------------
NOARCHIVELOG NO

SQL> select flashback_on from v$database;

FLASHBACK_ON
------------------
NO
```

> [!CAUTION]
> Changing **ARCHIVELOG** mode requires restarting the database.

You can make these changes using the following statements.

#### Restart the Database And Enable **ARCHIVELOG**

```
srvctl stop database -db cdba02_dc1
srvctl start instance -db cdba02_dc1 -instance cdba021 -startoption mount

sqlplus / as sysdba
alter database archivelog;
shutdown immediate
exit

srvctl start database -db cdba02_dc1
```

#### Enable **Force Logging** And **Flashback Database**

```
sqlplus / as sysdba
alter database force logging;
alter database flashback on;
exit
```

The current values on the primary database after the changes should be:

```
SQL> select log_mode, force_logging, flashback_on from v$database;

LOG_MODE     FORCE_LOGGING			     FLASHBACK_ON
------------ --------------------------------------- ------------------
ARCHIVELOG   YES				     YES
```

### Parameter STANDBY_FILE_MANAGEMENT

In most cases you would like to manage all files operations on the
standby database automatically:

    alter system set standby_file_management=auto;


### Create Standby Log Files

You have to add standby log files to the primary database. Every thread should have one more standby log file as online redo log files. The log file size must be the same.

#### Check Current Online Redo Logs

```
SQL> select thread# thr, group# grp, bytes from v$log order by 1,2;

       THR	  GRP	   BYTES
---------- ---------- ----------
	 1	    1  209715200
	 1	    2  209715200
	 2	    3  209715200
	 2	    4  209715200
```

So we have 2 threads with 2 online redo log files in each. 

#### Adding Standby Redo Logs

Based on the output above we need to create 3 standby redo log files to each thread.

> [!TIP]
> Before adding new standby redo log files check the parameter *DB_RECOVERY_FILE_DEST_SIZE* and ASM disk groups for enough space. The standby redo log files could be very large.

Adding standby log files to the first thread:

```
SQL> alter database add standby logfile thread 1 size 209715200;

Database altered.

SQL> r
  1* alter database add standby logfile thread 1 size 209715200

Database altered.

SQL> r
  1* alter database add standby logfile thread 1 size 209715200

Database altered.
```

Adding standby log files to the second thread:
```
SQL>  alter database add standby logfile thread 2 size 209715200;

Database altered.

SQL> r
  1*  alter database add standby logfile thread 2 size 209715200

Database altered.

SQL> r
  1*  alter database add standby logfile thread 2 size 209715200

Database altered.
```

Checking standby log files:
```
SQL> select thread# thr, group# grp, bytes from v$log order by 1,2;

       THR	  GRP	   BYTES
---------- ---------- ----------
	 1	    1  209715200
	 1	    2  209715200
	 2	    3  209715200
	 2	    4  209715200

SQL> select thread#, group#, bytes from v$standby_log order by 1, 2;

   THREAD#     GROUP#	   BYTES
---------- ---------- ----------
	 1	    5  209715200
	 1	    6  209715200
	 1	    7  209715200
	 2	    8  209715200
	 2	    9  209715200
	 2	   10  209715200

6 rows selected.
```

> [!NOTE]
> You can drop a standby log file using one of the following SQL statements:

- `alter database drop standby logfile member 'LogfilePath'`
- `alter database drop standby logfile group NN`

### Using Second Network

For large database with Data Guard you would usually use the second separated network. Such network should have a different SCAN name, SCAN listeners, VIP hosts and database listeners. 

#### Network Configuration for the Primary Database

| Network | Subnet | Public hosts | VIPs | SCAN <br> name | SCAN <br> Listener Port | Listener Port |
|---------|--------|--------------|------|----------------|-------------------|----------------|
| 1 | 192.168.4.0/22 | bol7rac2a, bol7rac2b | bol7rac2av, bol7rac2bv | bscan2 | 1521 | 1521 |
| 2 | 192.168.8.0/22 | bol7rac2a8, bol7rac2b8 | bol7rac2av8, bol7rac2bv8 | bscan2-dg | 1522 | 1522 |

#### Network Configuration for the Standby Database

| Network | Subnet | Public hosts | VIPs | SCAN <br> name | SCAN <br> Listener Port | Listener Port |
|---------|--------|--------------|------|----------------|-------------------|----------------|
| 1 | 192.168.4.0/22 | rkol7rac2a, rkol7rac2b | rkol7rac2av, rkol7rac2bv | rscan2 | 1521 | 1521 |
| 2 | 192.168.8.0/22 | rkol7rac2a8, rkol7rac2b8 | rkol7rac2av8, rkol7rac2bv8 | rscan2-dg | 1522 | 1522 |

#### Changing Listener Parameters

Here is an example SQL script to use both networks:
```
oracle@bol7rac2a> cat set_listener_networks.sql 
set echo on

alter system set local_listener='';
alter system set remote_listener='';
alter system set listener_networks='((name=net1)(local_listener=(address=(protocol=tcp)(host=bol7rac2av)(port=1521)))(remote_listener=bscan1:1521))','((name=net2)(local_listener=(address=(protocol=tcp)(host=bol7rac2av8)(port=1522)))(remote_listener=bscan2-dg:1522))' sid='cdba021';
alter system set listener_networks='((name=net1)(local_listener=(address=(protocol=tcp)(host=bol7rac2bv)(port=1521)))(remote_listener=bscan1:1521))','((name=net2)(local_listener=(address=(protocol=tcp)(host=bol7rac2bv8)(port=1522)))(remote_listener=bscan2-dg:1522))' sid='cdba022';
```

### INIT.ORA

The standby database parameters should be very similar to the database parameters on the primary database. You should use the *PFILE* of the primary database as a template for the standby database. 

```
oracle@bol7rac2a> mkdir -p /home/oracle/tests/data_guard/primary
oracle@bol7rac2a> cd /home/oracle/tests/data_guard/primary
oracle@bol7rac2a> sqlplus / as sysdba
...
SQL> create pfile='/home/oracle/tests/data_guard/primary/init.ora.primary' from spfile;

File created.
```

### Database Password File

The password file must be the same between the primary and standby databases. In RAC the password file is saved in ASM. You have to copy this file from the ASM to the local file system using *asmcmd*.

At first, you would run the following command as *oracle* OS user to get the ASM path of the database password file.

```
oracle@bol7rac2a> srvctl config database -db cdba02_dc1
Database unique name: cdba02_dc1
Database name: cdba02
Oracle home: /u01/oracle/rac19a
Oracle user: oracle
Spfile: +DATA/CDBA02_DC1/PARAMETERFILE/spfile.283.1205175523
Password file: +DATA/CDBA02_DC1/PASSWORD/pwdcdba02_dc1.269.1205174495
...
```

As *grid* OS user you should run the following *asmcmd* command using password file path from the above output:  
```
grid@bol7rac2a> asmcmd cp +DATA/CDBA02_DC1/PASSWORD/pwdcdba02_dc1.269.1205174495 /home/grid/cdba02_dc2.orapw
copying +DATA/CDBA02_DC1/PASSWORD/pwdcdba02_dc1.269.1205174495 -> /home/grid/cdba02_dc2.orapw
```

### Clearing Log Files In the Standby Database

Before starting the recovery apply on the standby database we would like to clear the online and standby log files on the newly created standby database. This would re-create them in ASM. We should generate the script for this task on the primary database.

```
set echo on pagesi 1000 linesi 255 trimsp on
spool clear_logfiles.log

select distinct 'alter database clear logfile group ' || group# || ';' from v$logfile order by 1;

spool off
```

The output will be written to the file *clear_logfiles.log*.

## Preparing the Standby Database

### Copy Files to Standby

#### INIT.ORA And Clear Log Files Script

As *oracle* OS user on the host *rkol7rac2a* (standby database):  

    oracle@rkol7rac2a> mkdir -p /home/oracle/tests/data_guard/standby


You should now copy the following files to the standby server *rkol7db1* 
directory */home/oracle/tests/data_guard/standby* from the directory
*/home/oracle/tests/data_guard/primary* on the primary server *bol7db1*:

- init.ora.primary
- clear_logfiles.log file with online and standby redo logs.

```
oracle@bol7rac2a> cd /home/oracle/tests/data_guard/primary
oracle@bol7rac2a> scp init.ora.primary rkol7rac2a:/home/oracle/tests/data_guard/standby
init.ora.primary                              100% 1953   591.4KB/s   00:00    
oracle@bol7rac2a> scp clear_logfiles.log rkol7rac2a:/home/oracle/tests/data_guard/standby
clear_logfiles.log                            100% 1123   812.1KB/s   00:00    
oracle@bol7rac1a> 
```

#### Database Password File

As *grid* OS user copy password file from *bol7rac2a* to *rkol7rac2a*:
```
grid@bol7rac2a> scp cdba02_dc2.orapw rkol7rac2a:
```

### Create Standby Database

Before doing the following steps, please check the environment variables 
ORACLE_HOME and ORACLE_BASE. 

### Prepare INIT.ORA

On the server *rkol7rac2a*:  

    cd /home/oracle/tests/data_guard/standby
    cp init.ora.primary init.ora.standby

In the file *init.ora.standby* you should check and maybe modify or comment out at least the following parameters:
  - audit_file_dest
  - cluster_interconnects: *Often you have to modify this parameter.*
  - control_files: *Comment out this parameter.*
  - db_create_file_dest
  - db_file_name_convert: *Usually you don't use it with ASM.*
  - db_recovery_file_dest
  - db_create_online_log_dest_xxx
  - db_unique_name=cdba02_dc2: *This parameter must be changed!*
  - listener_networks: *This parameter must be changed!*
  - local_listener: *This parameter must be changed!*
  - log_file_name_convert: *Usually you don't use it with ASM.*
  - pga_aggregate_target: *Do you have enough memory for a new standby database?*
  - remote_listener: *This parameter must be changed!*
  - sga_target: *Do you have enough memory for a new standby database?*

We are going to use ASM / Oracle managed files (OMF) on the standby database as well. That's why you usually don't need to set the parameters 
*db_file_name_convert* and *log_file_name_convert*. 

Please create directory for *AUDIT_FILE_DEST* on each server.
```
oracle@rkol7rac2a> mkdir -p /u01/oracle/admin/cdba02_dc2/adump
oracle@rkol7rac2a> ssh rkol7rac2b mkdir -p /u01/oracle/admin/cdba02_dc2/adump
```

### Create Cluster Resources for Standby Database

Using the output from `srvctl config database -db cdba02_dc1` on the primary database we can create the cluster resources for the new standby database. We will place all commands in the shell script *add_db.sh*.

```
oracle@rkol7rac2a> cat add_db.sh 
#!/bin/bash

DB_NAME=cdba02
DB_UNIQUE_NAME=${DB_NAME}_dc2

srvctl add database -db $DB_UNIQUE_NAME -dbname $DB_NAME \
  -oraclehome /u01/oracle/rac19c \
  -domain world \
  -dbtype RAC -role PHYSICAL_STANDBY \
  -diskgroup data,fra \

srvctl add instance -db $DB_UNIQUE_NAME -instance ${DB_NAME}1 -node rkol7rac2a
srvctl add instance -db $DB_UNIQUE_NAME -instance ${DB_NAME}2 -node rkol7rac2b
```

Run this script:
```
oracle@rkol7rac2a> bash -x ./add_db.sh 
+ DB_NAME=cdba02
+ DB_UNIQUE_NAME=cdba02_dc2
+ srvctl add database -db cdba02_dc2 -dbname cdba02 -oraclehome /u01/oracle/rac19c -domain world -dbtype RAC -role PHYSICAL_STANDBY -diskgroup data,fra
+ srvctl add instance -db cdba02_dc2 -instance cdba021 -node rkol7rac2a
+ srvctl add instance -db cdba02_dc2 -instance cdba022 -node rkol7rac2b
```

### Create SPFILE

You can start now the new instance in the *NOMOUNT* mode and create a new
*SPFILE* using the *init.ora.standby* file.

```
oracle@rkol7rac2a> export ORACLE_SID=cdba021
oracle@rkol7rac2a> sqlplus / as sysdba
...
SQL> startup nomount pfile='/home/oracle/tests/data_guard/standby/init.ora.standby'
ORACLE instance started.
...
SQL> create spfile='+DATA' from pfile='/home/oracle/tests/data_guard/standby/init.ora.standby';

File created.

SQL> shutdown abort
ORACLE instance shut down.
...
```

As *grid* OS user check the *SPFILE* name in ASM.

```
grid@rkol7rac2a> asmcmd ls -l +data/cdba02_dc2/parameterfile/spfile*
spfile.300.1205191177
...
```

Use this ASM path to add this file to the database cluster resources:
```
oracle@rkol7rac2a> srvctl modify database -db cdba02_dc2 -spfile +data/cdba02_dc2/PARAMETERFILE/spfile.300.1205191177
```

### Copy Database Password File

As *grid* OS user copy database password file to ASM disk group.

```
asmcmd pwcopy /home/grid/cdba02_dc2.orapw +data/cdba02_dc2/password/cdba02_dc2.orapw --dbuniquename cdba02_dc2
copying /home/grid/cdba02_dc2.orapw -> +data/cdba02_dc2/password/cdba02_dc2.orapw
ASMCMD-9453: failed to register password file as a CRS resource
grid@rkol7rac2a> 
```

ASM has created the file but could not register it in the cluster resources for the database. We can register it on our own as *oracle* OS user using the ASM file path from the *pwcopy* output:

```
srvctl modify database -db cdba02_dc2 -pwfile +data/cdba02_dc2/password/cdba02_dc2.orapw
```

### Restore Primary Database To the Standby Database

*Easy Connect Naming* allows us to run all commands without any new TNS aliases in the *tnsnames.ora*. The primary database runs on the RAC cluster with nodes *bol7rac1a* and *bol7rac1b*. We will use the VIP node on the first node for all restore tasks. We can find out the VIP node on the primary database using the following command:

```
oracle@bol7rac1a> srvctl config vip -node bol7rac1a
VIP exists: network number 1, hosting node bol7rac1a
VIP Name: bol7rac1av
VIP IPv4 Address: 192.168.5.32
VIP IPv6 Address: 
VIP is enabled.
VIP is individually enabled on nodes: 
VIP is individually disabled on nodes: 
oracle@bol7rac1a> 
```

The VIP name is *bol7rac1av*.

We will use RMAN feature *"Restoring Data Files Over the Network"* for the
RMAN RESTORE. In this case we don't have to customize any database network files.

#### Restore Control Files

Create standby control files with RMAN using service:

```
oracle@rkol7rac1a> cat restore_cf.rman 
connect target /
restore standby controlfile from service 'bol7rac1av/cdbrac_db1.world';
exit;
oracle@rkol7rac1a> rman @restore_cf.rman

Recovery Manager: Release 19.0.0.0.0 - Production on Wed May 28 20:58:29 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target *
2> restore standby controlfile from service 'bol7rac1av/cdbrac_db1.world';
3> exit;
connected to target database: CDBRAC (not mounted)

Starting restore at 28-MAY-25
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=393 instance=cdbrac1 device type=DISK
...
output file name=+DATA/CDBRAC_DB2/CONTROLFILE/current.398.1202331513
output file name=+FRA/CDBRAC_DB2/CONTROLFILE/current.692.1202331515
Finished restore at 28-MAY-25

Recovery Manager complete.
oracle@rkol7rac1a> 
```

#### Mount Standby Database

```
oracle@rkol7rac1a> sqlplus / as sysdba
...

SQL> alter database mount;

Database altered.

SQL> exit
```

#### Restore Database

Restore database with RMAN using service:

```
oracle@rkol7rac1a> cat restore_db.rman 
connect target /
run
{
  allocate channel t1 device type disk;
  allocate channel t2 device type disk;

  restore database from service 'bol7rac1av/cdbrac_db1.world'
    section size 256m;
}
exit;
oracle@rkol7rac1a> 

oracle@rkol7rac1a> rman @restore_db.rman

Recovery Manager: Release 19.0.0.0.0 - Production on Wed May 28 21:02:33 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target *
2> run
3> {
4>   allocate channel t1 device type disk;
5>   allocate channel t2 device type disk;
6> 
7>   restore database from service 'bol7rac1av/cdbrac_db1.world'
8>     section size 256m;
9> }
10> exit;
connected to target database: CDBRAC (DBID=687195340, not open)

using target database control file instead of recovery catalog
allocated channel: t1
channel t1: SID=20 instance=cdbrac1 device type=DISK

allocated channel: t2
channel t2: SID=146 instance=cdbrac1 device type=DISK

Starting restore at 28-MAY-25
Starting implicit crosscheck backup at 28-MAY-25
Crosschecked 1 objects
Finished implicit crosscheck backup at 28-MAY-25

Starting implicit crosscheck copy at 28-MAY-25
Finished implicit crosscheck copy at 28-MAY-25

searching for all files in the recovery area
cataloging files...
no files cataloged


channel t1: starting datafile backup set restore
channel t1: using network backup set from service bol7rac1av/cdbrac_db1.world
channel t1: specifying datafile(s) to restore from backup set
channel t1: restoring datafile 00001 to +DATA/CDBRAC_DB2/DATAFILE/system.399.1202331757
channel t1: restoring section 1 of 5
channel t2: starting datafile backup set restore
channel t2: using network backup set from service bol7rac1av/cdbrac_db1.world
channel t2: specifying datafile(s) to restore from backup set
channel t2: restoring datafile 00003 to +DATA/CDBRAC_DB2/DATAFILE/sysaux.400.1202331757
channel t2: restoring section 1 of 5
...
channel t2: restoring section 1 of 1
channel t1: restore complete, elapsed time: 00:00:03
channel t2: restore complete, elapsed time: 00:00:02
Finished restore at 28-MAY-25
released channel: t1
released channel: t2

Recovery Manager complete.
```

The restore performance and network load depend on the number of allocated 
channels and the value of *SECTION SIZE* parameter.


### Clear Log Files In the Standby Control Files

We have already prepared the file *clear_logfiles.log* with *ALTER DATABASE*
commands for all online and standby redo logs. Using this file we will prepare
and run the script *clear_logfiles.sql* to clear the redo logs.

> [!CAUTION]
> DO NOT RUN THIS SCRIPT ON THE PRIMARY DATABASE!!!
> MAKE SURE, YOU ARE CONNECTED TO THE NEW STANDBY DATABASE!!!

Original file from the primary database:
```
oracle@rkol7rac1a> cat clear_logfiles.log 
SQL> 
SQL> select distinct 'alter database clear logfile group ' || group# || ';' from v$logfile order by 1;

'ALTERDATABASECLEARLOGFILEGROUP'||GROUP#||';'
----------------------------------------------------------------------------
alter database clear logfile group 10;
alter database clear logfile group 1;
alter database clear logfile group 2;
alter database clear logfile group 3;
alter database clear logfile group 4;
alter database clear logfile group 5;
alter database clear logfile group 6;
alter database clear logfile group 7;
alter database clear logfile group 8;
alter database clear logfile group 9;

10 rows selected.

SQL> 
SQL> spool off
```

New created SQL file *clear_logfiles.sql* based on the *clear_logfiles.log* :

```
oracle@rkol7rac1a> cat clear_logfiles.sql
set echo on
alter database clear logfile group 10;
alter database clear logfile group 1;
alter database clear logfile group 2;
alter database clear logfile group 3;
alter database clear logfile group 4;
alter database clear logfile group 5;
alter database clear logfile group 6;
alter database clear logfile group 7;
alter database clear logfile group 8;
alter database clear logfile group 9;
oracle@rkol7rac1a> 
```

Run this SQL script on the standby database:
```
oracle@rkol7rac1a> sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Wed May 28 21:13:32 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0

SQL> @clear_logfiles
SQL> alter database clear logfile group 10;

Database altered.

SQL> alter database clear logfile group 1;

Database altered.
...
Database altered.

SQL> exit
```

### Restart Database

The database is restored. Now we should restart the database in *MOUNT* mode. Both instances must be running.

```
oracle@rkol7rac1a> srvctl stop database -db cdbrac_db2
oracle@rkol7rac1a> srvctl start database -db cdbrac_db2 -startoption mount
oracle@rkol7rac1a> srvctl status database -db cdbrac_db2
Instance cdbrac1 is running on node rkol7rac1a
Instance cdbrac2 is running on node rkol7rac1b
oracle@rkol7rac1a> 
```

## Create New Data Guard Broker Configuration

### Enable Data Guard Broker in INIT.ORA

#### Primary Database

```
oracle@bol7rac1a> cat set_dg.sql
set echo on
alter system set dg_broker_config_file1='+data/cdbrac_db1/dgconfig.file1';
alter system set dg_broker_config_file2='+fra/cdbrac_db1/dgconfig.file2';
alter system set dg_broker_start=true;

oracle@bol7rac1a> sqlplus / as sysdba
...
SQL> @set_dg
SQL> set echo on
SQL> alter system set dg_broker_config_file1='+data/cdbrac_db1/dgconfig.file1';

System altered.

SQL> alter system set dg_broker_config_file2='+fra/cdbrac_db1/dgconfig.file2';

System altered.

SQL> alter system set dg_broker_start=true;

System altered.

```

#### Standby Database

```
oracle@rkol7rac1a> cat set_dg.sql 
set echo on
alter system set dg_broker_config_file1='+data/cdbrac_db2/dgconfig.file1';
alter system set dg_broker_config_file2='+fra/cdbrac_db2/dgconfig.file2';
alter system set dg_broker_start=true;

oracle@rkol7rac1a> sqlplus / as sysdba
...
SQL> @set_dg 
SQL> alter system set dg_broker_config_file1='+data/cdbrac_db2/dgconfig.file1';

System altered.

SQL> alter system set dg_broker_config_file2='+fra/cdbrac_db2/dgconfig.file2';

System altered.

SQL> alter system set dg_broker_start=true;

System altered.

```


### Set up Data Guard Broker On the Primary Database

We use again *Easy Connect Naming* for database connect identifiers. We use RAC SCAN names and database unique names with domains.

```
oracle@bol7rac1a> cat create_config.sh 
dgmgrl / "create configuration cdbrac_dg as primary database is cdbrac_db1 connect identifier is 'bscan1:1521/cdbrac_db1.world'"
dgmgrl / "show configuration"
dgmgrl / "add database cdbrac_db2 as connect identifier is 'rscan1:1521/cdbrac_db2.world'"
dgmgrl / "show configuration"
dgmgrl / "enable configuration"
dgmgrl / "show configuration"

oracle@bol7rac1a> bash -x ./create_config.sh 
+ dgmgrl / 'create configuration cdbrac_dg as primary database is cdbrac_db1 connect identifier is '\''bscan1:1521/cdbrac_db1.world'\'''
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:31:10 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.
Configuration "cdbrac_dg" created with primary database "cdbrac_db1"
+ dgmgrl / 'show configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:31:15 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.

Configuration - cdbrac_dg

  Protection Mode: MaxPerformance
  Members:
  cdbrac_db1 - Primary database

Fast-Start Failover:  Disabled

Configuration Status:
DISABLED

+ dgmgrl / 'add database cdbrac_db2 as connect identifier is '\''rscan1:1521/cdbrac_db2.world'\'''
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:31:15 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.
Database "cdbrac_db2" added
+ dgmgrl / 'show configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:31:24 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.

Configuration - cdbrac_dg

  Protection Mode: MaxPerformance
  Members:
  cdbrac_db1 - Primary database
    cdbrac_db2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
DISABLED

+ dgmgrl / 'enable configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:31:24 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.
Enabled.
+ dgmgrl / 'show configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:31:45 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.

Configuration - cdbrac_dg

  Protection Mode: MaxPerformance
  Members:
  cdbrac_db1 - Primary database
    Warning: ORA-16905: The member was not enabled yet.

    cdbrac_db2 - Physical standby database 
      Error: ORA-16810: multiple errors or warnings detected for the member

Fast-Start Failover:  Disabled

Configuration Status:
ERROR   (status updated 34 seconds ago)

oracle@bol7rac1a> dgmgrl / "show configuration"
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:32:22 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.

Configuration - cdbrac_dg

  Protection Mode: MaxPerformance
  Members:
  cdbrac_db1 - Primary database
    Warning: ORA-16905: The member was not enabled yet.

    cdbrac_db2 - Physical standby database 
      Error: ORA-16810: multiple errors or warnings detected for the member

Fast-Start Failover:  Disabled

Configuration Status:
ERROR   (status updated 71 seconds ago)
```

The last error occurs because the Data Guard Broker is not running. Usually it takes some minutes before the databases are in sync.

```
oracle@bol7rac1a> dgmgrl / "show configuration"
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:33:03 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.

Configuration - cdbrac_dg

  Protection Mode: MaxPerformance
  Members:
  cdbrac_db1 - Primary database
    cdbrac_db2 - Physical standby database 
      Warning: ORA-16857: member disconnected from redo source for longer than specified threshold

Fast-Start Failover:  Disabled

Configuration Status:
WARNING   (status updated 38 seconds ago)
```

We can speed up this process archiving the online redo logs on the primary database.
```
oracle@bol7rac1a> sqlplus / as sysdba
...
SQL> alter system archive log current;

System altered.

SQL> exit
...

oracle@bol7rac1a> dgmgrl / "show configuration"
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed May 28 21:33:50 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdbrac_db1"
Connected as SYSDG.

Configuration - cdbrac_dg

  Protection Mode: MaxPerformance
  Members:
  cdbrac_db1 - Primary database
    cdbrac_db2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 28 seconds ago)

oracle@bol7rac1a> 
```

### Set Up Max. Availability Protection Mode

Run this commands in Data Guard Broker CLI:

```
edit database cdbrac_db1 set property LogXptMode=sync;
edit database cdbrac_db2 set property LogXptMode=sync;

edit configuration set protection mode as maxavailability;

show configuration;
```

### Run Switchover

Connect to the primary or standby database using listener. Don't use `dgmgrl /`, because the database needs to re-connect after restarting the local database.

```
oracle@bol7rac1a> dgmgrl 
...
DGMGRL> connect sys@bscan1:1521/cdbrac_db1.world      
Password:
Connected to "cdbrac_db1"
Connected as SYSDBA.
```

Run some checks before starting switchover.

```
DGMGRL> validate static connect identifier for all;
Oracle Clusterware on database "cdbrac_db1" is available for database restart.

Oracle Clusterware on database "cdbrac_db2" is available for database restart.


DGMGRL> show configuration;

Configuration - cdbrac_dg

  Protection Mode: MaxAvailability
  Members:
  cdbrac_db1 - Primary database
    cdbrac_db2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 26 seconds ago)

DGMGRL> show configuration lag;

Configuration - cdbrac_dg

  Protection Mode: MaxAvailability
  Members:
  cdbrac_db1 - Primary database
    cdbrac_db2 - Physical standby database 
                 Transport Lag:      0 seconds (computed 0 seconds ago)
                 Apply Lag:          0 seconds (computed 0 seconds ago)

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 30 seconds ago)
```

Run switchover.

```
DGMGRL> switchover to cdbrac_db2;
Performing switchover NOW, please wait...
Operation requires a connection to database "cdbrac_db2"
Connecting ...
Connected to "cdbrac_db2"
Connected as SYSDBA.
New primary database "cdbrac_db2" is opening...
Oracle Clusterware is restarting database "cdbrac_db1" ...
Connected to "cdbrac_db1"
Switchover succeeded, new primary is "cdbrac_db2"
DGMGRL> 
```

