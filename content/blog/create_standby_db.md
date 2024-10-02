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
select * from v$logfile where type = 'STANDBY' order by group#, member;
```

You can drop a standby logfile with one of the following statements:
- `alter database drop standby logfile member 'LogfilePath'`;
- `alter database drop standby logfile group NN;

### Network files:

listener.ora
```
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = rkol7db2)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = REGISTER))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME=db11a)
      (ORACLE_HOME=/u01/oracle/db19a)
      (GLOBAL_DBNAME=db11a_db1_DGMGRL.world)
    )
  )

```

tnsnames.ora
```
db11a_rman =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL=TCP)(HOST=rkol7db2)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SID = db11a)
    )
  )

```


## Create standby database

### Prepare init.ora and network files.

 - Copy pfile from primary to standby
 - Re-check at least the following parameters:
    - audit_file_dest
    - db_create_file_dest
    - db_recovery_file_dest
    - db_create_online_log_dest_x
    - db_unique_name
    - log_file_name_convert
    - cluster_interconnects
 - Comment out control files.
 - Create adit_file_dest directory

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
