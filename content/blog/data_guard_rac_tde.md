+++
title = 'Encrypt RAC Databases in Data Guard Configuration'
date = 2025-07-27T21:08:55+02:00
draft = true
summary = """
A practical guide to encrypting existing Oracle RAC databases with TDE in a Data Guard setup.
"""
tags = ["rac", "standby", "data guard", "tde"]
+++

## Encrypt RAC Database in Data Guard Configuration

### Example configuration

There are 2 RAC clusters consisting of the nodes *bol7rac2a*/*bol7rac2b* and *rkol7rac2a*/*rkol7rac2b*. The OS user "grid" was used to install the Grid Infrastructure.

The Oracle Release 19c RAC databases were installed using the OS user "oracle". The primary RAC database *cdba03* runs on the servers *bol7rac1* and *bol7rac2* and uses ASM. The database unique name is *cdba03_dc1*. The database domain is *world*. The standby RAC database *cdba03_dc2* runs on the servers *rkol7rac2a* and *rkol7rac2b*.

The Data Guard Broker configuration is created and enabled:
```
DGMGRL> show configuration;

Configuration - cdba03_dg

  Protection Mode: MaxPerformance
  Members:
  cdba03_dc1 - Primary database
    cdba03_dc2 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 52 seconds ago)
```

## Set Static TDE Database Parameters

### Prepare TDE Wallet Directory

In this blog we want to use TDE wallet for managing encryption keys. According to the Oracle documentation [About Using Transparent Data Encryption with Oracle Real Application Clusters](https://docs.oracle.com/en/database/oracle/oracle-database/19/asoag/using-transparent-data-encryption-with-other-oracle-features.html#GUID-49C5A375-3558-4678-922E-9C811511CF25) RAC database should use shared TDE wallet. The possible options are either ASM or ACFS. In this blog I want to use ACFS. 

In my previous blog [How To Create ACFS Mount Point?]({{< relref "create_acfs.md" >}}) I have already provided the basic steps. At the end of this blog the ACFS mount point */cfg* was created. The directory */cfg/oracle* with read / write permissions for *oracle* OS user was created and tested. We will use the directory */cfg/oracle/cdba03* for *WALLET_ROOT* database parameter.

As *oracle* OS user on *bol7rac2a*:
```
oracle@bol7rac2a> ls -ld /cfg/oracle
drwxr-xr-x 2 oracle oinstall 20480 27. Jul 20:19 /cfg/oracle

oracle@bol7rac2a> mkdir /cfg/oracle/cdba03
oracle@bol7rac2a> 

oracle@bol7rac2a> chmod 700 /cfg/oracle/cdba03
```

As *oracle* OS user on *rkol7rac2a*:
```
oracle@rkol7rac2a> ls -ld /cfg/oracle
drwxr-xr-x 2 oracle oinstall 20480 27. Jul  17:02 /cfg/oracle

oracle@rkol7rac2a> mkdir /cfg/oracle/cdba03
oracle@rkol7rac2a> 

oracle@rkol7rac2a> chmod 700 /cfg/oracle/cdba03
```

### Default Tablespace Encryption Algorithm

The default tablespace encryption in Oracle Release 19c is *AES128*. We want to use modern *AES256*. We can change the default tablespace encryption using the hidden parameter *_TABLESPACE_ENCRYPTION_DEFAULT_ALGORITHM*.

### Default Tablespace Encryption 

We will use default value *MANUAL_ENABLE*. This enables manual control which tablespaces are encrypted.

### Set Static Database Parameters

> [!Note]
> You have to restart the database after changing these parameters. 

#### Primary Database

Set database parameters:
```
oracle@bol7rac2a> sqlplus / as sysdba
...
SQL> alter system set "_tablespace_encryption_default_algorithm"='AES256' scope=spfile;

System altered.

SQL> alter system set tablespace_encryption=manual_enable scope=spfile;

System altered.

SQL> alter system set wallet_root='/cfg/oracle/cdba03' scope=spfile;
...
```

Restart the database:
```
oracle@bol7rac2a> srvctl stop database -db cdba03_dc1

oracle@bol7rac2a> srvctl start database -db cdba03_dc1
```

#### Standby Database

Set database parameters:
```
oracle@bol7rac2a> sqlplus / as sysdba
...
SQL> alter system set "_tablespace_encryption_default_algorithm"='AES256' scope=spfile;

System altered.

SQL> alter system set tablespace_encryption=manual_enable scope=spfile;

System altered.

SQL> alter system set wallet_root='/cfg/oracle/cdba03' scope=spfile;
...
```

Restart the database:
```
oracle@rkol7rac2a> srvctl stop database -db cdba03_dc2

oracle@rkol7rac2a> srvctl start database -db cdba03_dc2
```

### United or Isolated Mode?

We can use TDE encryption in multitenant environment in united or isolated mode:
- United mode uses one keystore in CDB root and other PDBs. The non-CDB databases use the same mode.
- Isolated mode enables using different keystores and master ncryption keys for different PDBs.

In this blog the *united* mode will be used.

