+++
title = 'How to Add a Standby Database?'
date = 2024-10-02T21:40:38+02:00
draft = true
summary = """
There are diffent ways to create and use a standby database. This 
document describes creating a new standby database using RMAN feature
restoring files over the network. The new standby database will 
be added to the Data Guard Broker configuration.
"""
+++

The example commands use the following configuration:
- The primary single instance database *cdbfa* runs on the server *rkol7db1*. This
database uses Oracle Managed Files (OMF) for all database files. The database unique name is *cdbfa_db1*. The database domain is *world*.
- The standby database should run on the server *rkol7db2* and use OMF as well.

## Preparing the Primary Database

### Set up Primary Database

You have to enable **ARCHIVELOG** mode on the primary database. To avoid 
possible issues during standby recovery you have to set **FORCE LOGGING** 
as well. You can check this using the following SELECT statement:

    select log_mode, force_logging from v$database;


You can set these parameters using the following statements:

```
shutdown immediate
alter database mount
alter database archivelog
shutdown immediate
startup
alter database force logging;
```

It's also recommended to use *flashback* especially on the standby 
database. 

```
select flashback_on from v$database;
...
alter database flashback on;
```

The current values on the primary database are :
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

### Parameter STANDBY_FILE_MANAGEMENT

In most cases you would like to manage all files operations on the
standby database automatically:

    alter system set standby_file_management=auto;


### Create Standby Logfiles

You have to add standby logfiles to the primary database. Every thread 
should have one more logfile.

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
- `alter database drop standby logfile member 'LogfilePath'`;
- `alter database drop standby logfile group NN;

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

Now you should the new entry using corresponing *ORACLE_SID*, 
*ORACLE_HOME* and the database *GLOBAL_NAME* to the *listener.ora* file.

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

#### TNSNAMES.ORA
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

### INIT.ORA ###

Now you should create a *PFILE* from the database *SPFILE*. You can use this 
file as a template for the standby database *PFILE*.

```
oracle@rkol7db1> mkdir -p /home/oracle/tests/data_guard/primary
oracle@rkol7db1> cd /home/oracle/tests/data_guard/primary
oracle@rkol7db1> sqlplus / as sysdba
...
SQL> create pfile='/home/oracle/tests/data_guard/primary/init.ora.primary' from spfile;

File created.
```

### Copy Files to Standby.

You should now copy the following files to the standby server directory 
*/home/oracle/tests/data_guard/standby*:
- init.ora.primary
- database password file from *$ORACLE_HOME/dbs*

For RAC/ASM you could use *asmcmd cp* command to copy the password file 
from ASM to the local file system.

## Create Standby Database

### Prepare INIT.ORA

You should copy the *init.ora.primary* file to *init.ora.standby* file. 
In this file you should check at least the following parameters:
  - audit_file_dest
  - db_create_file_dest
  - db_recovery_file_dest
  - db_create_online_log_dest_x
  - db_unique_name=cdbfa_db2
  - log_file_name_convert
  - cluster_interconnects
  - Comment out control files (*control_file*).

Please check, if you have to create these directories.

#### Single Instance

Now you can start the instance in the *NOMOUNT* mode and create a new
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

### Restore primary database to the standby database

Create standby control files with RMAN using service:
```
connect target /
restore standby controlfile from service 'rkol7db2/acdb16_db2.oracle.com';
exit;
```

Mount standby database: `alter database mount;`

Restore database with RMAN using service:
```
connect target /
run
{
  allocate channel t1 device type disk;
  allocate channel t2 device type disk;
  allocate channel t3 device type disk;

  restore database from service 'rkol7db2/acdb16_db2.oracle.com' 
    section size 64m;
}
exit;
```

## Set up Data Guard Broker on the primary database

### Clear logfiles in the standby control file.
- Clear all standby logfiles. This would re-create both standby and online OMF logfiles:
```
set echo on pagesi 1000 linesi 255 trimsp on

select distinct 'alter database clear logfile group ' || group# || ';' from v$logfile order by 1;
```

### Create new Data Guard Configuration.


```
dgmgrl / "create configuration acdb16_dg as primary database is acdb16_db2 connect identifier is 'rkol7db2:1521/acdb16_db2.oracle.com'"
dgmgrl / "show configuration"
dgmgrl / "add database acdb16_db3 as connect identifier is 'rkol7db3:1521/acdb16_db3.oracle.com'"
dgmgrl / "show configuration"
dgmgrl / "enable configuration"
dgmgrl / "show configuration"
```

Set up max. availability protection mode:

```
edit database Primary_Database set property LogXptMode=sync;
edit database Standby_Database set property LogXptMode=sync;

edit configuration set protection mode as maxavailability;
```

## Set up listener entries for both database in listener.ora

Primary:
```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME=acdb16)
      (ORACLE_HOME=/u01/oracle/db19c)
      (GLOBAL_DBNAME=acdb16_db2_DGMGRL.oracle.com)
    )
  )
```

Standby:
```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME=acdb16)
      (ORACLE_HOME=/u01/oracle/db19c)
      (GLOBAL_DBNAME=acdb16_db3_DGMGRL.oracle.com)
    )
  )
```

Run switchover:
```
dgmgrl
connect sys/oracle@rkol7db2/acdb16_db2.oracle.com as sysdba
validate static connect identifier for all;
show configuration;
switchover to acdb16_db3;
```
