+++
title = 'How To Add a Large Standby Database Fast'
date = 2025-06-30T18:45:35+02:00
draft = false
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
> - [Optimizing RMAN Restore from Primary to Standby Database](#optimizing-rman-restore-from-primary-to-standby-database)
> - [Rolling Forward a Physical Standby Database](#rolling-forward-a-physical-standby-database)


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

### Prepare INIT.ORA

On the server *rkol7rac2a*:  

    cd /home/oracle/tests/data_guard/standby
    cp init.ora.primary init.ora.standby

In the file *init.ora.standby* you should check and maybe modify or comment out at least the following parameters:
  - audit_file_dest: *Don't forget to create these directories!*
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

## Create Standby Database

Before doing the following steps, please check the environment variables 
ORACLE_HOME and ORACLE_BASE. 

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

Check the *SPFILE* output in *srvctl config database -db cdba02_dc2* command. Usually it should be set. If not, as *grid* OS user check the *SPFILE* name in ASM.

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
```
*pwcopy* has copied the file but could not register it in the cluster resources for the database. 
```
copying /home/grid/cdba02_dc2.orapw -> +data/cdba02_dc2/password/cdba02_dc2.orapw
ASMCMD-9453: failed to register password file as a CRS resource
grid@rkol7rac2a> 
```
We can register it on our own as *oracle* OS user using the ASM file path from the *pwcopy* output:

As *grid* OS user:
```
grid@rkol7rac2a> asmcmd ls -l +data/cdba02_dc2/password/
Type      Redund  Striped  Time             Sys  Name
PASSWORD  UNPROT  COARSE   JUL 01 12:00:00  N    cdba02_dc2.orapw => +DATA/CDBA02_DC2/PASSWORD/pwdcdba02_dc2.270.1205326127
PASSWORD  UNPROT  COARSE   JUL 01 12:00:00  Y    pwdcdba02_dc2.270.1205326127
```

We need the file *pwdcdba02_dc2.270.1205326127* and not the alias. As *oracle* OS user:
```
oracle@rkol7rac2a> srvctl modify database -db cdba02_dc2 -pwfile +data/cdba02_dc2/password/pwdcdba02_dc2.270.1205326127
```

### Optimizing RMAN Restore from Primary to Standby Database

Instead of using RMAN backups we want to use the RMAN feature [Restoring and Recovering Files Over the Network](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/rman-recovery-advanced.html#GUID-8E64485B-3788-4389-A892-8C560F12443F). 

We assume that the primary database is large. Your network must be fast for the following steps. (If not you can use the *RESTORE* commands from my previous blog [How to Add a Standby RAC Database?]({{< relref "create_standby_rac_db.md" >}})). 

If you run the RMAN command from just one node, that node can use the SCAN name to fetch data from all source nodes. However, if your network is fast, this single target node running RMAN may become a performance bottleneck. In such cases, it's better to both read and write RMAN data using all RAC nodes in parallel.

We want to use the RMAN command *ALLOCATE CHANNEL* to allocate different RMAN channels to different RAC nodes. But in this case we have to specify connect string (user, password and TNS-alias). You should not save any password in the files, so we are going to use [Oracle Secure External Password Store for Password Credentials](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/configuring-authentication.html#GUID-3EA07020-A9F3-4FF9-9518-E1AEA3BDDBBE). 

The path to the Secure External Password Store (SEPS) will be specified in *sqlnet.ora*. We are going to set the TNS_ADMIN for this RAC database to use our own network configuration instead of the default *$ORACLE_HOME/network/admin* files.

#### Restore Control Files

At first, we are going to restore the standby control file using the primary database. We will use SCAN name on the second network (Data Guard Network) with the corresponding SCAN listener port 1522.

Start up the standby database and don't mount it.

```
oracle@rkol7rac2a> sqlplus / as sysdba
...
SQL> startup nomount
ORACLE instance started.

Total System Global Area 8589933432 bytes
Fixed Size		    9198456 bytes
Variable Size		 1426063360 bytes
Database Buffers	 7147094016 bytes
Redo Buffers		    7577600 bytes
SQL> exit
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.26.0.0.0
```

Create standby control file using the primary database control file:
```
oracle@rkol7rac2a> cat restore_cf.rman 
connect target /
restore standby controlfile from service 'bscan2-dg:1522/cdba02_dc1.world';
exit;

oracle@rkol7rac2a> rman @restore_cf.rman

Recovery Manager: Release 19.0.0.0.0 - Production on Wed Jul 2 16:32:25 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target *
2> restore standby controlfile from service 'bscan2-dg:1522/cdba02_dc1.world';
3> exit;
connected to target database: CDBA02 (not mounted)

Starting restore at 02-JUL-25
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=16 instance=cdba021 device type=DISK

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: using network backup set from service bscan2-dg:1522/cdba02_dc1.world
channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:02
output file name=+DATA/CDBA02_DC2/CONTROLFILE/current.270.1205425949
output file name=+FRA/CDBA02_DC2/CONTROLFILE/current.266.1205425949
Finished restore at 02-JUL-25

Recovery Manager complete.
oracle@rkol7rac2a> 
```

#### Set TNS_ADMIN

For our tests we would create new files *sqlnet.ora* and *tnsnames.ora*. We can place them in the directory */u01/oracle/net* on every node.

sqlnet.ora
```
sqlnet.wallet_override = true
wallet_location =
  (source =
    (method = file)
    (method_data =
      (directory = /u01/oracle/wallets/cdba02_dc2)
    )
  )
```

tnsnames.ora
```
cdba02_dc2_instance1 =
  (description=
    (address=(protocol=tcp)(host=rkol7rac2av8)(port=1522))
    (connect_data=(server=dedicated)(service_name=cdba02_dc2.world))
  )
cdba02_dc2_instance2 =
  (description=
    (address=(protocol=tcp)(host=rkol7rac2bv8)(port=1522))
    (connect_data=(server=dedicated)(service_name=cdba02_dc2.world))
  )
```

These are net service names for the local standby database. All of them use the second network (Data Guard / Backup network). We use the corresponding VIPs with the port number for the Data Guard database listener.

We have created these files on the first node. Now we have to copy them to another node:
```
oracle@rkol7rac2a> ssh rkol7rac2b mkdir -p /u01/oracle/net
oracle@rkol7rac2a> cd /u01/oracle/net
oracle@rkol7rac2a> scp * rkol7rac2b:$PWD
sqlnet.ora                                    100%  160   189.3KB/s   00:00    
tnsnames.ora                                  100%  368   615.8KB/s   00:00    

```

#### Create Secure External Password Store

The path to the new Secure External Password Store (SEPS) is already specified in the *sqlnet.ora* file. We are going to use the following commands:
```
mkstore -wrl /u01/oracle/wallets/cdba02_dc2 -create
mkstore -wrl /u01/oracle/wallets/cdba02_dc2 -createCredential ConnectString User Password
```

The wallet must be created on both standby nodes. We can repeat these steps on both nodes or just use a script.
```
oracle@rkol7rac2a> cat create_wallet.sh 
#!/usr/bin/bash 

WALLET_DIR=/u01/oracle/wallets/cdba02_dc2
WALLET_PASSWORD=OracleSEPS4321
SYS_PASSWORD=oracle

echo "Creating wallet directories..."

for host in rkol7rac2a rkol7rac2b ; do
  echo "Host: $host"
#  ssh $host rm -rf $WALLET_DIR
  ssh $host mkdir -p $WALLET_DIR
done

echo "Creating wallet on the local wallet..."
mkstore -wrl $WALLET_DIR -create <<ENDE
$WALLET_PASSWORD
$WALLET_PASSWORD
ENDE

echo "Adding credential to the local wallet..."
echo "$WALLET_PASSWORD" | mkstore -wrl $WALLET_DIR -createCredential cdba02_dc1_instance1 sys $SYS_PASSWORD
echo "$WALLET_PASSWORD" | mkstore -wrl $WALLET_DIR -createCredential cdba02_dc1_instance2 sys $SYS_PASSWORD


echo "List local wallet credentials..."
echo "$WALLET_PASSWORD" | mkstore -wrl $WALLET_DIR -listCredential 

echo "Copy wallet to another node..."
for host in rkol7rac2b ; do
  echo "Host: $host"
  scp -r $WALLET_DIR/* $host:$WALLET_DIR
done
```

Now we can execute the script to create SEPS on both nodes.


#### Mount Standby Database


Set *TNS_ADMIN* to the new location and check the first instance:
```
oracle@rkol7rac2a> export TNS_ADMIN=/u01/oracle/net

oracle@rkol7rac2a> tnsping cdba02_dc2_instance1

TNS Ping Utility for Linux: Version 19.0.0.0.0 - Production on 02-JUL-2025 16:39:58

Copyright (c) 1997, 2024, Oracle.  All rights reserved.

Used parameter files:
/u01/oracle/net/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (description= (address=(protocol=tcp)(host=rkol7rac2av8)(port=1522)) (connect_data=(server=dedicated)(service_name=cdba02_dc2.world)))
OK (0 msec)

```

Shut down the local instance.
```
sqlplus / as sysdba
...
SQL> shutdown abort
ORACLE instance shut down.
```

Start all RAC instances. We have restored the standby control file, so that we can use *-startoption mount* to mount the database.
```
oracle@rkol7rac2a> srvctl start database -db cdba02_dc2 -startoption mount

oracle@rkol7rac2a> srvctl status database -db cdba02_dc2
Instance cdba021 is running on node rkol7rac2a
Instance cdba022 is running on node rkol7rac2b

```

#### Change Snapshot Controlfile Location

The restored control file contains RMAN configuration from the primary database. At least the snapshot control file location must be changed. You should check other parameters using *show all* RMAN command.

```
racle@rkol7rac2a> rman target /

Recovery Manager: Release 19.0.0.0.0 - Production on Tue Jul 1 23:30:41 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to target database: CDBA02 (DBID=2429009365, not open)

RMAN> show all;
...
RMAN> CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+fra/cdba02_dc2/snapshot.cf';
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+fra/cdba02_dc2/snapshot.cf';
new RMAN configuration parameters:
CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+fra/cdba02_dc2/snapshot.cf';
new RMAN configuration parameters are successfully stored
```

#### Clear Log Files In the Standby Control Files

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

#### Restore Database

Now we can restore the primary database using all channels on both nodes.

> [!TIP]
> Don't forget to set NLS_DATE_FORMAT before starting RMAN. This would set RMAN date format for RMAN job output:\
> `export NLS_DATE_FORMAT="dd-mon-yy hh24:mi:ss"`

> [!TIP]
> Use RMAN *SET COMMAND ID TO 'text' to set the string into the V$SESSION.CLIENT_INFO column of all channels. You could use this information to determine which database server sessions correspond to which RMAN channels.

```
oracle@rkol7rac2a> cat restore_db.rman 
connect target /
run {

# Set V$SESION.CLIENT_INFO column
  set command id to 'RESTORE CDBA02_DC2';

# 2 channels from the first node
  allocate channel d1 device type disk connect "/@cdba02_dc2_instance1 as sysdba";
  allocate channel d2 device type disk connect "/@cdba02_dc2_instance1 as sysdba";

# 2 channels from the first node
  allocate channel d3 device type disk connect "/@cdba02_dc2_instance2 as sysdba";
  allocate channel d4 device type disk connect "/@cdba02_dc2_instance2 as sysdba";

# Set section size depending on the number of channels, RAC nodes and the database size.
  restore database 
    from service 'bscan2-dg:1522/cdba02_dc1.world'
    section size 64G;
}
exit
oracle@rkol7rac2a> 
```

The restore would take some time. Usually we would start such RMAN restore operation using UNIX *nohup* command.

```
oracle@rkol7rac2a> nohup rman @restore_db.rman &
[1] 18196
oracle@rkol7rac2a> nohup: ignoring input and appending output to 'nohup.out'
...
oracle@rkol7rac2a> tail -f nohup.out
channel d2: SID=146 instance=cdba021 device type=DISK

allocated channel: d3
channel d3: SID=274 instance=cdba022 device type=DISK

allocated channel: d4
channel d4: SID=397 instance=cdba022 device type=DISK

Starting restore at 07-jul-25 21:09:22
Starting implicit crosscheck backup at 07-jul-25 21:09:22
Crosschecked 1 objects
Crosschecked 1 objects
Finished implicit crosscheck backup at 07-jul-25 21:09:22

Starting implicit crosscheck copy at 07-jul-25 21:09:22
Finished implicit crosscheck copy at 07-jul-25 21:09:22

searching for all files in the recovery area
cataloging files...
no files cataloged


channel d1: starting datafile backup set restore
channel d1: using network backup set from service bscan2-dg:1522/cdba02_dc1.world
channel d1: specifying datafile(s) to restore from backup set
channel d1: restoring datafile 00001 to +DATA/CDBA02_DC2/DATAFILE/system.291.1205874565
channel d1: restoring section 1 of 1
channel d2: starting datafile backup set restore
...
channel d4: restoring section 1 of 1
channel d3: restore complete, elapsed time: 00:00:02
channel d4: restore complete, elapsed time: 00:00:02
channel d1: restore complete, elapsed time: 00:00:08
channel d2: restore complete, elapsed time: 00:00:08
Finished restore at 07-jul-25 21:10:09
released channel: d1
released channel: d2
released channel: d3
released channel: d4

Recovery Manager complete.
```

The restore performance and network load depend on the number of allocated channels, RAC nodes and the value of *SECTION SIZE* parameter. The channel allocation output confirmed, that we have used both instances for restore:
```
allocated channel: d1
channel d1: SID=393 instance=cdba021 device type=DISK <== Instance 1

allocated channel: d2
channel d2: SID=145 instance=cdba021 device type=DISK <== Instance 1

allocated channel: d3
channel d3: SID=143 instance=cdba022 device type=DISK <== Instance 2

allocated channel: d4
channel d4: SID=269 instance=cdba022 device type=DISK <== Instance 2

```

Now we can unset TNS_ADMIN variable: `unset TNS_ADMIN`

####  Rolling Forward a Physical Standby Database

The database restore could take some time. During this time a lot of online redo logs could be produced on the primary database. We can speed up this process using the RMAN feature [Rolling Forward a Physical Standby Database Using the RECOVER Command](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/rman-recovery-advanced.html#GUID-93C8F7F1-E1FB-485D-ABC7-860757B24A98)

The RMAN *RECOVER* command would read all incremented blocks since the last restore database command. Before running the RMAN *RECOVER* we have to make sure, that only instance is running.

```
oracle@rkol7rac2a> srvctl stop database -db cdba02_dc2 -stopoption abort

oracle@rkol7rac2a> srvctl start instance -db cdba02_dc2 -instance cdba021 -startoption mount

oracle@rkol7rac2a> cat recover_db.rman 
connect target /
run {
  allocate channel d1 device type disk;
  allocate channel d2 device type disk;
  allocate channel d3 device type disk;
  allocate channel d4 device type disk;

  recover standby database 
    from service 'bscan2-dg:1522/cdba02_dc1.world';
}
exit

oracle@rkol7rac2a> nohup rman @recover_db.rman &
[1] 18145

tail -f nohup.out
...
Recovery Manager: Release 19.0.0.0.0 - Production on Wed Jul 2 17:42:20 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target *
...
contents of Memory Script:
{
   restore standby controlfile from service  'bscan2-dg:1522/cdba02_dc1.world';
   alter database mount standby database;
}
executing Memory Script
...
starting media recovery

media recovery complete, elapsed time: 00:00:00
Finished recover at 02-JUL-25
Executing: alter system set standby_file_management=auto
Finished recover at 02-JUL-25

Recovery Manager complete.
```


#### Clear Log Files In the Standby Control Files 

The RMAN *RECOVERY* command re-created the control files from the primary database. The log file must be cleared again. We can just use the existing script *clear_logfiles.sql* from the previous step.

> [!CAUTION]
> DO NOT RUN THIS SCRIPT ON THE PRIMARY DATABASE!!!
> MAKE SURE, YOU ARE CONNECTED TO THE NEW STANDBY DATABASE!!!

### Restart Database

The database is restored. Now we should restart the database in *MOUNT* mode. Both instances must be running.

```
oracle@rkol7rac2a> srvctl stop database -db cdba02_dc2

oracle@rkol7rac2a> srvctl start database -db cdba02_dc2 -startoption mount

oracle@rkol7rac2a> srvctl status database -db cdba02_dc2

oracle@rkol7rac2a> srvctl status database -db cdba02_dc2
Instance cdba021 is running on node rkol7rac2a
Instance cdba022 is running on node rkol7rac2b
oracle@rkol7rac2a> 

```

## Create New Data Guard Broker Configuration

### Enable Data Guard Broker in INIT.ORA

#### Primary Database

```
oracle@bol7rac2a> cat set_dg.sql 
set echo on
alter system set dg_broker_config_file1='+data/cdba02_dc1/dgconfig.file1';
alter system set dg_broker_config_file2='+fra/cdba02_dc1/dgconfig.file2';
alter system set dg_broker_start=true;


oracle@bol7rac1a> sqlplus / as sysdba
...
SQL> @set_dg
SQL> alter system set dg_broker_config_file1='+data/cdba02_dc1/dgconfig.file1';

System altered.

SQL> alter system set dg_broker_config_file2='+fra/cdba02_dc1/dgconfig.file2';

System altered.

SQL> alter system set dg_broker_start=true;

System altered.
```

#### Standby Database

```
oracle@rkol7rac2a> cat set_dg.sql 
set echo on
alter system set dg_broker_config_file1='+data/cdba02_dc2/dgconfig.file1';
alter system set dg_broker_config_file2='+fra/cdba02_dc2/dgconfig.file2';
alter system set dg_broker_start=true;

oracle@rkol7rac1a> sqlplus / as sysdba
...
SQL> @set_dg 
QL> alter system set dg_broker_config_file1='+data/cdba02_dc2/dgconfig.file1';

System altered.

SQL> alter system set dg_broker_config_file2='+fra/cdba02_dc2/dgconfig.file2';

System altered.

SQL> alter system set dg_broker_start=true;

System altered.
```


### Set up Data Guard Broker On the Primary Database

We use again *Easy Connect Naming* for database connect identifiers. We use RAC SCAN names and database unique names with domains.

```
oracle@bol7rac2a> cat create_dg_config.sh 
dgmgrl / "create configuration cdba02_dg as primary database is cdba02_dc1 connect identifier is 'bscan2-dg:1522/cdba02_dc1.world'"
dgmgrl / "show configuration"
dgmgrl / "add database cdba02_dc2 as connect identifier is 'rscan2-dg:1522/cdba02_dc2.world'"
dgmgrl / "show configuration"
dgmgrl / "enable configuration"
dgmgrl / "show configuration"

oracle@bol7rac2a> bash -x ./create_dg_config.sh 
...
```

Usually it takes some minutes before the databases are in sync.

```
oracle@bol7rac2a> dgmgrl / "show configuration"
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Wed Jul 2 18:08:45 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdba02_dc1"
Connected as SYSDG.

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 48 seconds ago)
```

### Set Up Max. Availability Protection Mode

Run this commands in Data Guard Broker CLI:

```
edit database cdba02_dc1 set property LogXptMode=sync;
edit database cdba02_dc2 set property LogXptMode=sync;

edit configuration set protection mode as maxavailability;

show configuration;
```

### Run Switchover

Connect to the primary or standby database using listener. Don't use `dgmgrl /`, because the database needs to re-connect after restarting the local database.

```
oracle@bol7rac2a> dgmgrl 
...
DGMGRL> connect sys@bscan2-dg:1522/cdba02_dc1.world
Password:
Connected to "cdba02_dc1"
Connected as SYSDBA.
```

Run some checks before starting switchover.

```
DGMGRL> validate static connect identifier for all;
Oracle Clusterware on database "cdba02_dc1" is available for database restart.

Oracle Clusterware on database "cdba02_dc2" is available for database restart.

DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxAvailability
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 44 seconds ago)

DGMGRL> show configuration lag;

Configuration - cdba02_dg

  Protection Mode: MaxAvailability
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 
                 Transport Lag:      0 seconds (computed 1 second ago)
                 Apply Lag:          0 seconds (computed 1 second ago)

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 54 seconds ago)

```

Run switchover.

```
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

