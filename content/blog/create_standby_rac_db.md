+++
title = 'How to Add a Standby RAC Database?'
date = 2025-05-13T09:35:09+02:00
draft = true
summary = """
There are different ways to create and use a standby database. This document describes creating a new standby RAC database using RMAN. The new standby database will be added to the Data Guard Broker configuration.
"""
tags = ["rac", "standby", "rman", "data guard"]
+++


The example commands use the following configuration.

There are 2 RAC clusters consisting of the nodes *bol7rac1a*/*bol7rac1b* and *rkol7rac1a*/*rkol7rac1b*. The OS user "grid" was used to install the Grid Infrastructure. There are 2 ASM disk groups DATA and FRA on each cluster.

The RAC databases were installed using the OS user "oracle". The primary RAC database *cdbrac* runs on the servers *bol7rac1* and *bol7rac2* and uses ASM. The database unique name is *cdbrac_db1*. The database domain is *world*.

```
SQL> show parameter db_name

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_name 			     string	 cdbfa
SQL> show parameter db_unique_name

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_unique_name			     string	 cdbfa_db1
SQL> show parameter domain

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
db_domain			     string	 world
SQL> 
```

The standby database should run on the servers *rkol7rac1* and *rkol7rac2*. The database unique name will be *cdbrac_db2*.

## Preparing the Primary Database

### Set up Primary Database

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
srvctl stop database -db cdbrac_db1
srvctl start instance -db cdbrac_db1 -instance cdbrac1 -startoption mount

sqlplus / as sysdba
alter database archivelog;
shutdown immediate
exit

srvctl start database -db cdbrac_db1
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

### INIT.ORA

The standby database parameters should be very similar to the database parameters on the primary database. You should use the *PFILE* of the primary database as a template for the standby database. 

```
oracle@bol7rac1a> mkdir -p /home/oracle/tests/data_guard/primary
oracle@bol7rac1a> cd /home/oracle/tests/data_guard/primary
oracle@bol7rac1a> sqlplus / as sysdba
...
SQL> create pfile='/home/oracle/tests/data_guard/primary/init.ora.primary' from spfile;

File created.
```

### Database Password File

The password file must be the same between the primary and standby databases. In RAC the password file is saved in ASM. You have to copy this file from the ASM to the local file system using *asmcmd*.

At first, you would run the following command as *oracle* OS user to get the ASM path of the database password file.

```
oracle@bol7rac1a> srvctl config database -db cdbrac_db1
Database unique name: cdbrac_db1
Database name: cdbrac
Oracle home: /u01/oracle/rac19b
Oracle user: oracle
Spfile: +DATA/CDBRAC_DB1/PARAMETERFILE/spfile.362.1201599925
Password file: +DATA/CDBRAC_DB1/PASSWORD/pwdcdbrac_db1.346.1201598869
...
```

As *grid* OS user you should run the following *asmcmd* command using password file path from the above output:  
```
grid@bol7rac1a> asmcmd cp +DATA/CDBRAC_DB1/PASSWORD/pwdcdbrac_db1.346.1201598869 /home/grid/orapwdcdbrac
copying +DATA/CDBRAC_DB1/PASSWORD/pwdcdbrac_db1.346.1201598869 -> /home/grid/orapwdcdbrac
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

As *oracle* OS user on the host *rkol7db1*:  

    oracle@rkol7db2> mkdir -p /home/oracle/tests/data_guard/standby


You should now copy the following files to the standby server *rkol7db1* 
directory */home/oracle/tests/data_guard/standby* from the directory
*/home/oracle/tests/data_guard/primary* on the primary server *bol7db1*:

- init.ora.primary
- clear_logfiles.log file with online and standby redo logs.

```
oracle@rkol7db1> cd /home/oracle/tests/data_guard/primary
oracle@rkol7db1> scp init.ora.primary rkol7db2:/home/oracle/tests/data_guard/standby
init.ora.primary                                                  100% 1253     1.5MB/s   00:00    

oracle@rkol7db1> scp clear_logfiles.log rkol7db2:/home/oracle/tests/data_guard/standby
clear_logfiles.log                                                100%  543   652.1KB/s   00:00    

oracle@rkol7db1> scp $ORACLE_HOME/dbs/orapwcdbfa rkol7db2:/home/oracle/tests/data_guard/standby
orapwcdbfa                                                        100% 2048     3.0MB/s   00:00    
```

For RAC/ASM you could use *asmcmd cp* command to copy the password file 
from ASM to the local file system.

#### Database Password File

As *grid* OS user 
## Create Standby Database

Before doing the following steps, please check the environment variables 
ORACLE_HOME and ORACLE_BASE. 

### Prepare INIT.ORA

On the server *rkol7db2*:  

    cd /home/oracle/tests/data_guard/standby
    cp init.ora.primary init.ora.standby

In the file *init.ora.standby* you should check and maybe modify at least 
the following parameters:
  - audit_file_dest
  - db_create_file_dest
  - db_file_name_convert
  - db_recovery_file_dest
  - db_create_online_log_dest_xxx
  - db_unique_name=cdbfa_db2
  - log_file_name_convert
  - cluster_interconnects
  - Comment out control files (*control_file*).

We are going to use Oracle managed files (OMF) on the standby database as 
well. That's why you usually don't need to set the parameters 
*db_file_name_convert* and *log_file_name_convert*. 

Please check, if you have to create the corresponding directories:
- audit_file_dest
- db_create_file_dest
- db_recovery_file_dest

These are the modified parameters in *init.ora.standby* for the new standby
database *cdbfa_db2*:

```
*.audit_file_dest='/u01/oracle/admin/cdbfa_db2/adump'
...
#*.control_files='/u01/oracle/databases/db19/CDBFA_DB1/controlfile/o1_mf_mhv7b60r_.ctl','/u01/oracle/databases/fra/CDBFA_DB1/controlfile/o1_mf_mhv7b64j_.ctl'
...
*.db_create_file_dest='/u01/oracle/databases/db19'
...
*.db_recovery_file_dest='/u01/oracle/databases/fra'
...
*.db_unique_name='cdbfa_db2'
```

In this case the folllowing directories must be created:
```
mkdir -p /u01/oracle/admin/cdbfa_db2/adump
mkdir -p /u01/oracle/databases/db19
mkdir -p /u01/oracle/databases/fra
```

#### Single Instance

You can start now the new instance in the *NOMOUNT* mode and create a new
*SPFILE* using the *init.ora.standby* file.
```
oracle@rkol7db2> cp orapwcdbfa $ORACLE_HOME/dbs
oracle@rkol7db2> export ORACLE_SID=cdbfa
sracle@rkol7db2> sqlplus / as sysdba
...
SQL> startup nomount pfile='/home/oracle/tests/data_guard/standby/init.ora.standby' 
...
SQL> create spfile from pfile='/home/oracle/tests/data_guard/standby/init.ora.standby';
File created.
SQL> shutdown abort
ORACLE instance shut down.
SQL> startup nomount
...
```

#### RAC

- RAC: Create database configuration (srvctl + asmcmd).
- Start instance using the pfile and create the spfile.
- Create or copy the password file.
- Startup instance in nomount mode

### Restore Primary Database To the Standby Database

Easy Connect Naming allows to run all commands without any new TNS aliases in 
the *tnsnames.ora*. 

We will use RMAN feature *"Restoring Data Files Over the Network"* for the
RMAN RESTORE.

Create standby control files with RMAN using service:

```
oracle@rkol7db2> cat restore_cf.rman 
connect target /
restore standby controlfile from service 'rkol7db1/cdbfa_db1.world';
exit;

oracle@rkol7db2> rman @restore_cf.rman
...
RMAN> connect target *
2> restore standby controlfile from service 'rkol7db1/cdbfa_db1.world';
3> exit;
connected to target database: CDBFA (not mounted)
...
output file name=/u01/oracle/databases/db19/CDBFA_DB2/controlfile/o1_mf_n239lhjw_.ctl
output file name=/u01/oracle/databases/fra/CDBFA_DB2/controlfile/o1_mf_n239lhno_.ctl
Finished restore at 12-MAY-25

Recovery Manager complete.
oracle@rkol7db2> 
```

Now we have to mount the standby database: 
```
sqlplus / as sysdba
alter database mount;
exit
```

Restore database with RMAN using service:

```
oracle@rkol7db2> cat restore_db.rman 
connect target /
run
{
  allocate channel t1 device type disk;
  allocate channel t2 device type disk;

  restore database from service 'rkol7db1/cdbfa_db1.world'
    section size 256m;
}
exit;


oracle@rkol7db2> rman @restore_db.rman

Recovery Manager: Release 19.0.0.0.0 - Production on Mon May 12 09:46:40 2025
Version 19.24.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

RMAN> connect target *
2> run
3> {
4>   allocate channel t1 device type disk;
5>   allocate channel t2 device type disk;
6> 
7>   restore database from service 'rkol7db1/cdbfa_db1.world'
8>     section size 256m;
9> }
10> exit;
connected to target database: CDBFA (DBID=898928678, not open)

using target database control file instead of recovery catalog
allocated channel: t1
channel t1: SID=12 device type=DISK

allocated channel: t2
channel t2: SID=258 device type=DISK

Starting restore at 12-MAY-25
Starting implicit crosscheck backup at 12-MAY-25
Finished implicit crosscheck backup at 12-MAY-25

Starting implicit crosscheck copy at 12-MAY-25
Finished implicit crosscheck copy at 12-MAY-25

searching for all files in the recovery area
cataloging files...
no files cataloged


channel t1: starting datafile backup set restore
channel t1: using network backup set from service rkol7db1/cdbfa_db1.world
channel t1: specifying datafile(s) to restore from backup set
channel t1: restoring datafile 00001 to /u01/oracle/databases/db19/CDBFA_DB2/datafile/o1_mf_system_n239v22y_.dbf
channel t1: restoring section 1 of 6
channel t2: starting datafile backup set restore
channel t2: using network backup set from service rkol7db1/cdbfa_db1.world
channel t2: specifying datafile(s) to restore from backup set
channel t2: restoring datafile 00003 to /u01/oracle/databases/db19/CDBFA_DB2/datafile/o1_mf_sysaux_n239v24y_.dbf
channel t2: restoring section 1 of 4
...
channel t1: restoring datafile 00008 to /u01/oracle/databases/db19/CDBFA_DB2/238481C02E841B13E0630107A8C01C2B/datafile/o1_mf_undotbs1_n239wxqf_.dbf
channel t1: restoring section 2 of 2
channel t1: restore complete, elapsed time: 00:00:01
channel t2: restore complete, elapsed time: 00:00:04
Finished restore at 12-MAY-25
released channel: t1
released channel: t2

Recovery Manager complete.
```

The restore performance and network load depend on the number of allocated 
channels and the value of SECTION SIZE parameter.


### Clear logfiles in the standby control file.

We have already prepared the file *clear_logfiles.log* with *ALTER DATABASE*
commands for all online and standby redo logs. Using this file we can prepare
and ran the script *clear_logfiles.sql* to clear the redo logs.

**DO NOT RUN THIS SCRIPT ON THE PRIMARY DATABASE. MAKE SURE, YOU ARE CONNECTED 
TO THE NEW STANDBY DATABASE!!!**

Original file from the primary database:
```
oracle@rkol7db2> cat clear_logfiles.log
SQL> 
SQL> select distinct 'alter database clear logfile group ' || group# || ';' from v$logfile order by 1;

'ALTERDATABASECLEARLOGFILEGROUP'||GROUP#||';'
----------------------------------------------------------------------------
alter database clear logfile group 1;
alter database clear logfile group 2;
alter database clear logfile group 3;
alter database clear logfile group 4;
alter database clear logfile group 5;
alter database clear logfile group 6;
alter database clear logfile group 7;

7 rows selected.

SQL> 
SQL> spool offexit
```

New created SQL file based on the *clear_logfiles.log* :

```
oracle@rkol7db2> cat clear_logfiles.sql
set echo on

alter database clear logfile group 1;
alter database clear logfile group 2;
alter database clear logfile group 3;
alter database clear logfile group 4;
alter database clear logfile group 5;
alter database clear logfile group 6;
alter database clear logfile group 7;
```

Running this SQL file on the standby database:
```
oracle@rkol7db2> sqlplus / as sysdba
...
SQL> @clear_logfiles
SQL> 
SQL> alter database clear logfile group 1;

Database altered.
...
SQL> alter database clear logfile group 7;

Database altered.

SQL> exit
```

## Create New Data Guard Broker Configuration

### Set init.ora Parameters

Set Data Guard database parameters on both primary and standby databases:

```
alter system set dg_broker_config_file1='/u01/oracle/db19f/dbs/cdbfa_dg.dat1';
alter system set dg_broker_config_file2='/u01/oracle/db19f/dbs/cdbfa_dg.dat2';
alter system set dg_broker_start=true;
```

### Set up Data Guard Broker On the Primary Database

```
dgmgrl / "create configuration cdbfa_dg as primary database is cdbfa_db1 connect identifier is 'rkol7db1:1521/cdbfa_db1.world'"
dgmgrl / "show configuration"
dgmgrl / "add database cdbfa_db2 as connect identifier is 'rkol7db2:1521/cdbfa_db2.world'"
dgmgrl / "show configuration"
dgmgrl / "enable configuration"
dgmgrl / "show configuration"
```

### Set Up Max. Availability Protection Mode

Run this commands in Data Guard Broker CLI:

```
edit database cdbfa_db1 set property LogXptMode=sync;
edit database cdbfa_db2 set property LogXptMode=sync;

edit configuration set protection mode as maxavailability;

show configuration;
```

### Run Switchover

Connect to the primary or standby database using listener. Don't use `dgmgrl /`, because the database needs to re-connect after restarting the local database.

```
dgmgrl
connect sys/oracle@rkol7db1/cdbfa_db1.world
```

Run some checks before starting switchover:

```
validate static connect identifier for all
show configuration lag
show configuration
````

Run switchover:
```
switchover to cdbfa_db2
```
