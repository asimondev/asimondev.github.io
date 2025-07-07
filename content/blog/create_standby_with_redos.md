+++
title = 'How To Create a Standby With Redo Transport'
date = 2025-07-07T21:29:38+02:00
draft = false
summary = """
During creating a new standby database, you have at first restore or duplicate the current primary database. During this time interval the archived redo logs could be removed by usual database backup job. This blog describes how to use Data Guard broker to transport archived redo logs during the creation of the standby database.
"""
tags = ["rac", "standby", "rman", "data guard"]
+++

In my previous blog [How To Add a Large Standby Databse Fast]({{< relref "fast_create_standby_rac_db.md" >}}) I have provided the basic steps. This article adds some new steps.

Usually you would restore the primary database. Then you would create a new Data Guard configuration and add both primary and standby databases to this configuration. Data Guard broker allows set up a redo transport between the primary database and another *external destination*. The following changes starts before running RMAN *RESTORE DATABASE FROM SERVICE* command.

## Create Data Guard Broker configuration.

### Add Data Guard Broker Database Parameters

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

oracle@rkol7rac2a> sqlplus / as sysdba
...
SQL> @set_dg.sql
SQL> alter system set dg_broker_config_file1='+data/cdba02_dc2/dgconfig.file1';

System altered.

SQL> alter system set dg_broker_config_file2='+fra/cdba02_dc2/dgconfig.file2';

System altered.

SQL> alter system set dg_broker_start=true;

System altered.
```


### Set up Data Guard Broker On the Primary Database

We use again *Easy Connect Naming* for database connect identifiers. We use RAC SCAN names and database unique names with domains. The new Data Guard configuration will consist only of the primary database.

Create a Data Guard Script:
```
oracle@bol7rac2a> cat create_dg_config_primary.sh 
dgmgrl / "create configuration cdba02_dg as primary database is cdba02_dc1 connect identifier is 'bscan2-dg:1522/cdba02_dc1.world'"
dgmgrl / "show configuration"
dgmgrl / "enable configuration"
dgmgrl / "show configuration"
```

Run this script on the primary database:
```
oracle@bol7rac2a> bash -x ./create_dg_config_primary.sh 
+ dgmgrl / 'create configuration cdba02_dg as primary database is cdba02_dc1 connect identifier is '\''bscan2-dg:1522/cdba02_dc1.world'\'''
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Mon Jul 7 21:55:02 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdba02_dc1"
Connected as SYSDG.
Configuration "cdba02_dg" created with primary database "cdba02_dc1"
+ dgmgrl / 'show configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Mon Jul 7 21:55:06 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdba02_dc1"
Connected as SYSDG.

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database

Fast-Start Failover:  Disabled

Configuration Status:
DISABLED

+ dgmgrl / 'enable configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Mon Jul 7 21:55:06 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdba02_dc1"
Connected as SYSDG.
Enabled.
+ dgmgrl / 'show configuration'
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Mon Jul 7 21:55:10 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdba02_dc1"
Connected as SYSDG.

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    Warning: ORA-16905: The member was not enabled yet.

Fast-Start Failover:  Disabled

Configuration Status:
WARNING   (status updated 7 seconds ago)

oracle@bol7rac2a> 
```

### Add New Standby Databsae as External Destinantion

We have not got a running standby database. But we have already restored the standby control file and mounted this database. We will add this *future* standby database as a new external destination to the primary database.

```
oracle@bol7rac2a> cat add_ext_dest.dg 
edit configuration set property ExternalDestination1='SERVICE="rscan2-dg:1522/cdba02_dc2.world" db_unique_name=cdba02_dc2';
show configuration;
oracle@bol7rac2a> 

oracle@bol7rac2a> dgmgrl -echo /
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Mon Jul 7 22:04:15 2025
Version 19.26.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "cdba02_dc1"
Connected as SYSDG.
DGMGRL> @add_ext_dest.dg
@add_ext_dest.dg
edit configuration set property ExternalDestination1='SERVICE="rscan2-dg:1522/cdba02_dc2.world" db_unique_name=cdba02_dc2';
Property "externaldestination1" updated
show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - External destination 1

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 11 seconds ago)

DGMGRL> exit
exit
```

### Checking External Destination

We should now create some archived redo logs on the primary database:
```
sqlplus / as sysdba
...
SQL> alter system archive log current;

System altered.

SQL> r
  1* alter system archive log current

System altered.

SQL> r
  1* alter system archive log current

System altered.

SQL> 
```

On the standby database site in ASM we should find new archvied redo logs:

```
grid@rkol7rac2a> asmcmd ls -l +fra/cdba02_dc2/archivelog/2025_07_07
Type        Redund  Striped  Time             Sys  Name
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_71.265.1205877869
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_72.262.1205878133
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_73.280.1205878193
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_56.263.1205877869
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_57.271.1205878133
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_58.281.1205878193
grid@rkol7rac2a> 
grid@rkol7rac2a> 
grid@rkol7rac2a> 
grid@rkol7rac2a> asmcmd ls -l +fra/cdba02_dc2/archivelog/2025_07_07
Type        Redund  Striped  Time             Sys  Name
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_71.265.1205877869
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_72.262.1205878133
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_73.280.1205878193
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_1_seq_74.283.1205878251
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_56.263.1205877869
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_57.271.1205878133
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_58.281.1205878193
ARCHIVELOG  UNPROT  COARSE   JUL 07 22:00:00  Y    thread_2_seq_59.282.1205878251
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

### RMAN RESTORE / RECOVERY Database

Now we can run other RMAN commands (*RESTORE DATABASE* and maybe *RECOVER STANDBY DATABASE*) as in the previous blog.

### Add Standby to Data Guard Configuration

We have to remove the *ExternalDestination1* property from the Data Guard configuration. Then we can immediately add the standby database. 

Create a new Data Guard script *add_standby.dg".
```
oracle@bol7rac2a> cat add_standby.dg 
show configuration;
edit configuration set property ExternalDestination1='';
add database cdba02_dc2 as connect identifier is 'rscan2-dg:1522/cdba02_dc2.world';
enable configuration;
show configuration;
oracle@bol7rac2a> 
```

Run this script:
```
oracle@bol7rac2a> dgmgrl -echo /
...
DGMGRL> @add_standby.dg
@add_standby.dg
show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - External destination 1

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 48 seconds ago)

edit configuration set property ExternalDestination1='';
Property "externaldestination1" updated
add database cdba02_dc2 as connect identifier is 'rscan2-dg:1522/cdba02_dc2.world';
Database "cdba02_dc2" added
enable configuration;
Enabled.
show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 
      Error: ORA-16810: multiple errors or warnings detected for the member

Fast-Start Failover:  Disabled

Configuration Status:
ERROR   (status updated 72 seconds ago)

DGMGRL> exit
exit
```

The new standby database was added. After a while the configuration status should change to *SUCCESS*.