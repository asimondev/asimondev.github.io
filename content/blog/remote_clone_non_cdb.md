+++
title = 'How To Migrate Non-CDB 19c Database to PDB?'
date = 2024-09-29T16:48:17+02:00
draft = false
summary = """
You can use Remote Clone and Refreshable Clone to create a new PDB
from the existing non-CDB database in a few steps. The corresponding SQL 
statement would copy in parallel datafiles from the non-CDB database 
directly to the CDB database. The script noncdb_to_pdb.sql helps 
to convert the existing database data dictionary from non-CDB to PDB."""
+++

## Create Migration User on Non-CDB Database.

You have to create a new user or use an existing user with the
specific privileges. For instance, you would create a new user
**noncdbadmin** for the migration. You can drop this user after 
migration.

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

## Create Database Link from CDB Root Container to Non-CDB Database

```
conn / as sysdba

define my_link='my_non_cdb'
define my_non_cdb_user='noncdbadmin'
define my_pwd='andrej'
-- Use Easy Connect string or TNS alias.
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

Both *SELECTs* at the end should return **valid** results!

Now you are ready to create a new PDB from the existing non-CDB database. 
You could use one of these two methods:
- Cloning a remote non-CDB database to PDB with one SQL statement;
- Create a refreshable clone of the remote non-CDB dataase at first. In the next
step you would refresh the clone before starting the data dictionary migration.

You would usually use just *remote clone*, if the non-CDB database is small
or you can afford to set this database to *read only* for long time. 

The usage of *refreshable clone* helps to reduce the downtime window. During 
the creating of PDB as *refreshable clone*, the non-CDB database can stay 
opened in *read write* mode as usual. Only the second step (*refresh*) will 
be executed in *read only* mode.

## Cloning a Remote Non-CDB

We use *remote clone* multitenant feature to clone the existing
non-CDB database into the existing CDB as a new PDB. 

You must use the database name and not the database unique name for 
the non-CDB database name in the *FROM* clause together with the 
database link name.

### Set non-CDB database in READ ONLY mode

#### Single Instance

```
shutdown immediate
startup open read only
select open_mode from v$database;
```

#### RAC

```
srvctl stop database -db ga
srvctl start instance -db ga -instance ga1 -startoption "read only"
```

### Create Pluggable Database

```
conn / as sysdba

define my_pdb=pdbg01
-- Use database name (g01) and not database unique name!
define my_non_cdb=g01@my_non_cdb

set echo on

create pluggable database &my_pdb from &my_non_cdb
/
```

The created PDB will be in MOUNT status: `show pdbs`

Check PDB_PLUG_IN_VIOLATIONS:

```
select * from pdb_plug_in_violations order by time;
select * from pdb_plug_in_violations where name='PDB_Name' order by time;
```

## Cloning a Remote Non-CDB using Refreshable Clone PDB

During the cloning the source database can be uses as usual.

```
conn / as sysdba

define my_pdb=pdbg01
-- Use database name (g01) and not database unique name!
define my_non_cdb=g01@my_non_cdb

set echo on

create pluggable database &my_pdb from &my_non_cdb refresh mode manual
/
```

The created PDB will be in MOUNT status: `show pdbs`

Check PDB_PLUG_IN_VIOLATIONS:

```
select * from pdb_plug_in_violations order by time;
select * from pdb_plug_in_violations where name='PDBName' order by time;
```

Now you can refresh the created PDB. Before doing that you should set 
the source database in the *READ ONLY* mode (see above). Don't open this PDB!

```
conn / as sysdba
-- define my_pdb=pdbg01
alter pluggable database &my_pdb refresh
/
```

After *REFRESH* you must disable this PDB as *Refreshable Clone PDB*.

```
conn / as sysdba

-- define my_pdb=pdbg01

set echo on

alter pluggable database &my_pdb refresh mode none
/
```

## Check PDB Database Parameters

Some of parameters will be copied from non-CDB database. Other parameters
will be set using values from the CDB$ROOT.


Reset PDB parameters:
```
conn / as sysdba

alter session set container=...
show parameter ...
alter system reset ...
alter system reset ... scope=spfile

select name, type, value, display_value from v$system_parameter where con_id = 8
order by name
/

select name from v$system_parameter where ISPDB_MODIFIABLE='TRUE'
order by name
/

```
### Run noncdb_to_pdb.sql Script

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
conn / as sysdba
select * from pdb_plug_in_violations order by time;
select * from pdb_plug_in_violations where name='PDBName' order by time;
```

### Open New PDB

```
conn / as sysdba
alter pluggable databsae ... open instances=all;
```

### Possible Next Steps

#### Database Services

You have to delete the services on the non-CDB database and create 
them on the new PDB. Don't forget to link the services to the new PDB!

#### PDB State

If you want to open the new PDB automatically, you have to start 
this PDB and save the current state.

```
conn / as sysdba
alter pluggable database PDB_Name open instances=all;
alter pluggable database all save state instances=all;
```

## Troubleshooting

### Clean up Old Rows from PDB_PLUG_IN_VIOLATIONS

```
conn / as sysdba
exec dbms_pdb.clear_plugin_violations('PDB_Name');
select * from pdb_plug_in_violations where name='PDB_Name' order by time;
```

### Check Installed Database Options

```
set echo on

set linesi 80 pagesi 20
col comp_id for a10
col comp_name for a35
col status for a10
col version for a10

select comp_id, comp_name, status, version from dba_registry order by 1;
```

### Install Missing Database Options in PDB

Sometimes you have to install missing options in PDB, because they 
are already installed in CDB. Below some examples for installing
often missing database options.

- APS: @?/olap/admin/cataps.sql
- XOQ: @?/olap/admin/catxoq.sql
- Context: @?/ctx/admin/catctx.sql ctxsys SYSAUX TEMP NOLOCK

### How to Drop PDB?

```
conn / as sysdba
alter pluggable database PDB_Name close immediate instances=all;
drop pluggable database PDB_Name including datafiles;
```