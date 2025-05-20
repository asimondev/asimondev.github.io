+++
title = 'How to Add a Single Instance Standby Database?'
date = 2024-10-02T21:40:38+02:00
draft = false
summary = """
There are diffent ways to create and use a standby database. This 
document describes creating a new single instance standby database using RMAN. The new standby database will 
be added to the Data Guard Broker configuration.
"""
+++

The example commands use the following configuration.

Oracle RDBMS software is installed into the ORACLE_HOME */u01/oracle/db19f* on both servers.

The primary single instance database *cdbfa* runs on the server *rkol7db1*. This database uses Oracle Managed Files (OMF) for all database files. The database unique name is *cdbfa_db1*. The database domain is *world*.

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

The standby database should run on the server *rkol7db2* and use OMF as well. The database unique name will be *cdbfa_db2*.

## Preparing the Primary Database

### Set up Primary Database

You have to enable **Archivelog** mode on the primary database. To avoid possible issues during standby recovery you have to set **Force Logging** as well. It's recommended to use **Flashback Database** especially on the standby database. 

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

You can set these parameters using the following statements.

#### Restart the Database And Enable **Archivelog**

```
shutdown immediate
startup mount
alter database archivelog;
shutdown immediate
startup
```

#### Enable **Force Logging** And **Flashback Database**

```
alter database force logging;
alter database flashback on;
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


### Create Standby Logfiles

You have to add standby logfiles to the primary database. Every thread 
should have one more standby logfile as online redo logfile.

```
select thread# thr, group# grp, bytes from v$log order by 1,2;
...
alter database add standby thread ... logfile ... size ...
...
select thread#, group#, bytes from v$standby_log order by 1, 2;
```

So will will run on the primary database:
```
SQL> select thread# thr, group# grp, bytes from v$log order by 1,2;

       THR	  GRP	   BYTES
---------- ---------- ----------
	 1	    1  209715200
	 1	    2  209715200
	 1	    3  209715200

SQL> alter database add standby logfile size 209715200;
Database altered.
...
SQL> select thread#, group#, bytes from v$standby_log order by 1, 2;

   THREAD#     GROUP#	   BYTES
---------- ---------- ----------
	 0	    4  209715200
	 0	    5  209715200
	 0	    6  209715200
	 0	    7  209715200
```

You can drop a standby logfile with one of the following statements:
- `alter database drop standby logfile member 'LogfilePath'`
- `alter database drop standby logfile group NN`

### Network files:

#### LISTENER.ORA

Usually on RAC you don't have to add any entry. But this example is for
the single instance database. If you don't have working *listener.ora* and
*sqlnet.ora* files, you can create them using the **netca** utility:

```
oracle@rkol7db1> cd $ORACLE_HOME/network/admin

oracle@rkol7db1> netca -silent -responsefile $ORACLE_HOME/assistants/netca/netca.rsp 

Parsing command line arguments:
    Parameter "silent" = true
    Parameter "responsefile" = /u01/oracle/db19f/assistants/netca/netca.rsp
Done parsing command line arguments.
Oracle Net Services Configuration:
Profile configuration complete.
Oracle Net Listener Startup:
    Running Listener Control: 
      /u01/oracle/db19f/bin/lsnrctl start LISTENER
    Listener Control complete.
    Listener started successfully.
Listener configuration complete.
Oracle Net Services configuration successful. The exit code is 0
```

Now you should add the new entry using corresponing *ORACLE_SID*, 
*ORACLE_HOME* and the database *GLOBAL_NAME* values to the 
*listener.ora* file.

```
...
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME=cdbfa)
      (ORACLE_HOME=/u01/oracle/db19f)
      (GLOBAL_DBNAME=cdbfa_db1_DGMGRL.world)
    )
  )
...
```
Use `lsnrctl reload` to load the new configuration to the listener. 

You should use the *listener.ora* and *sqlnet.ora* on the primary database as 
a template to create corresponding files on the standby database. 

This is a *listener.ora* file on the new standby database host *rkol7db2*:

```
# listener.ora Network Configuration File: /u01/oracle/db19f/network/admin/listener.ora
# Generated by Oracle configuration tools.

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = rkol7db2)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME=cdbfa)
      (ORACLE_HOME=/u01/oracle/db19f)
      (GLOBAL_DBNAME=cdbfa_db2_DGMGRL.world)
    )
  )

```

The differences to the primary database are:
- Hostname *rkol7db2* instead of *rkol7db1*
- Global database name *cdbfa_db2_DGMGRL.world* instead of *cdbfa_db1_DGMGRL.wolrd*

You have to start or reload the listener on the standby database now.

#### TNSNAMES.ORA

We want to use RMAN to duplicate the primary database to the standby database. 
We will add the corresponding RMAN TNS aliases to the *tnsnames.ora* file. 

```
cdbfa_db1_rman =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL=TCP)(HOST=rkol7db1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SID = cdbfa)
    )
  )

cdbfa_db2_rman =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL=TCP)(HOST=rkol7db2)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SID = cdbfa)
    )
  )

```

You should copy these TNS aliases to the standby database *tnsnames.ora* file.

### INIT.ORA ###

Now you should create a *PFILE* from the primary database *SPFILE*. You can use this file as a template for the standby database *PFILE*.

```
oracle@rkol7db1> mkdir -p /home/oracle/tests/data_guard/primary
oracle@rkol7db1> cd /home/oracle/tests/data_guard/primary
oracle@rkol7db1> sqlplus / as sysdba
...
SQL> create pfile='/home/oracle/tests/data_guard/primary/init.ora.primary' from spfile;

File created.
```

### Clearing Logfiles In the Standby Database.

Before starting the recovery apply on the standby database we would like to
clear the online and standby logfiles. This would re-create them in OMF
configuration. We can generate the script for this taks on the primary database.

```
set echo on pagesi 1000 linesi 255 trimsp on
spool clear_logfiles.log

select distinct 'alter database clear logfile group ' || group# || ';' from v$logfile order by 1;

spool off
```

### Copy Files to Standby.

On the host *rkol7db2*:  

    oracle@rkol7db2> mkdir -p /home/oracle/tests/data_guard/standby


You should now copy the following files to the standby server *rkol7db2* 
directory */home/oracle/tests/data_guard/standby* from the current directory
*/home/oracle/tests/data_guard/primary* on the primary server *rkol7db1*:

- init.ora.primary
- database password file from *$ORACLE_HOME/dbs*
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

> [!CAUTION]
> DO NOT RUN THIS SCRIPT ON THE PRIMARY DATABASE. MAKE SURE, YOU ARE CONNECTED TO THE NEW STANDBY DATABASE!!!

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
