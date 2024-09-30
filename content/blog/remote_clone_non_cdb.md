+++
title = 'How To Migrate Non-CDB 19c Database to PDB?'
date = 2024-09-29T16:48:17+02:00
draft = false
+++

## Create Migration User on Non-CDB Database

```
conn / as sysdba

define my_user='noncdbadmin'
define my_pwd='andrej'

set echo on

-- drop user &my_user;

create user &my_user identified by "&my_pwd"
/

grant create session, create pluggable database,
  sysoper to &my_user
/
```

## Creating a New CDB

At first you have to create a new CDB database or use the existing one.
You should select the corresponding database block size and the 
database character set. The database block size set must be the same. 
Usually you would use the CDB with the latest Unicode variable-width 
multibyte character set (AL32UTF8). This would allow you to 
migrate non-cdb databases and PDBs with other character sets as well.

### Create Database Link from CDB Root Container

```
conn / as sysdba

define my_link='my_non_cdb'
define my_non_cdb_user='noncdbadmin'
define my_pwd='andrej'
define my_tns='rkol7db1/g01.world'

set echo on

-- drop database link &my_link;

create database link &my_link
  connect to &my_non_cdb_user identified by "&my_pwd"
  using '&my_tns'
/

select * from global_name;
select * from global_name@&my_link;
```

Both SELECTs at the end should return **valid** results!

## Set non-CDB database in READ ONLY mode

### Single Instance

```
shutdown immediate
startup open read only
select open_mode from v$database;
```

### RAC

```
srvctl stop database -db ga
srvctl start instance -db ga -instance ga1 -startoption "read only"
```

## Create Pluggable Database

We use *Remote Clone* multitenant feature to clone the existing
non-CDB database into the existing CDB as a new PDB.

```
conn / as sysdba

define my_pdb=pdbg01
define my_non_cdb=g01@my_non_cdb

set echo on


create pluggable database &my_pdb from &my_non_cdb
/
```

The created PDB will be in MOUNT status.

Check PDB_PLUG_IN_VIOLATIONS:

```
select * from pdb_plug_in_violations order by time;
select * from pdb_plug_in_violations where name='PDBName' order by time;
```

Check PDB parameters:

```
conn / as sysdba
select name, con_uid from v$pdbs order by name;
select name, value$ from pdb_spfile$ where pdb_uid=...
order by name
/
```

Reset PDB parameters:
```
conn / as sysdba

alter session set container=...
alter system reset ...
alter system reset ... scope=spfile

select name, type, value, display_value from v$system_parameter where con_id = 8
order by name
/

select name from v$system_parameter where ISPDB_MODIFIABLE='TRUE'
order by name
/

```
### Run noncdb_to_pdb.sql script.

```
conn / as sysdba

define my_pdb=pdbg01

set echo on

alter session set container=&my_pdb
/

set echo off

@?/rdbms/admin/noncdb_to_pdb

show pdbs
```

Check PDB_PLUG_IN_VIOLATIONS:

```
select * from pdb_plug_in_violations order by time;
select * from pdb_plug_in_violations where name='PDBName' order by time;
```

### Open new PDB

```
conn / as sysdba
alter pluggable databsae ... open instances=all;
```

## Troubleshooting.

### Clean up old rows from PDB_PLUG_IN_VIOLATIONS

```
conn / as sysdba
exec dbms_pdb.clear_plugin_violations('PDB_Name');
commit;
select * from pdb_plug_in_violations where name='PDBName' order by time;
```

### Install database options.

Sometimes you have install missing options in PDB, because they are already 
installed in CDB.

- APS: @?/olap/admin/cataps.sql
- XOQ: @?/olap/admin/catxoq.sql
- Context: @?/ctx/admin/catctx.sql ctxsys SYSAUX TEMP NOLOCK
