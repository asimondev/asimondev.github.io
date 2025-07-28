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

> [!NOTE]
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

### Set TDE_ENCRYPTION Parameter

The *TDE_ENCRYPTION* database parameter set the type of used TDE keystore. We want to use TDE Wallet (wallet keystore). This is a dynamic database parameter. We can set this parameter on both database without restart.

Primary database:
```
oracle@bol7rac2a> sqlplus / as sysdba
...
SQL> show parameter wallet_root

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
wallet_root			     string	 /cfg/oracle/cdba03

SQL> alter system set tde_configuration="keystore_configuration=file";

System altered.

SQL> show parameter tde_configuration

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
tde_configuration		     string	 keystore_configuration=file
```

Standby database:
```
oracle@rkol7rac2a> sqlplus / as sysdba
...
SQL> show parameter wallet_root

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
wallet_root			     string	 /cfg/oracle/cdba03
SQL> alter system set tde_configuration="keystore_configuration=file";

System altered.

SQL> show parameter tde_configuration

NAME				     TYPE	 VALUE
------------------------------------ ----------- ------------------------------
tde_configuration		     string	 keystore_configuration=file
```

### Create TDE Wallet

At the moment we have set all parameters, but no TDE Wallet exists:
```
oracle@bol7rac2a> ls /cfg/oracle/cdba03/
oracle@bol7rac2a> 
```

We are going to create a new password-protected TDE wallet on the primary database. We will use the wallet password *HiddenPassword2233*.
```
SQL> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS			       WALLET_TYPE	    WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/cfg/oracle/cdba03/tde/
NOT_AVAILABLE		       UNKNOWN		    SINGLE    NONE     UNDEFINED
	 1

FILE

NOT_AVAILABLE		       UNKNOWN		    SINGLE    UNITED   UNDEFINED
	 2

FILE

NOT_AVAILABLE		       UNKNOWN		    SINGLE    UNITED   UNDEFINED
	 3
SQL> 

SQL> administer key management create keystore identified by "HiddenPassword2233";

keystore altered.

SQL> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS			       WALLET_TYPE	    WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/cfg/oracle/cdba03/tde/
CLOSED			       UNKNOWN		    SINGLE    NONE     UNDEFINED
	 1

FILE

CLOSED			       UNKNOWN		    SINGLE    UNITED   UNDEFINED
	 2

FILE

CLOSED			       UNKNOWN		    SINGLE    UNITED   UNDEFINED
	 3


SQL> 
```

The new directory *tde* with the wallet was created below the *WALLET_ROOT* directory:
```
oracle@bol7rac2a> ls -l /cfg/oracle/cdba03/tde/
total 4
-rw------- 1 oracle asmadmin 2553 28. Jul  21:32 ewallet.p12
oracle@bol7rac2a> 
```

It's recommended to use auto-login software keystore for RAC databases with Data Guard configuration. It's not allowed to use local auto-login software keystore for RAC databases.

```
SQL> administer key management create auto_login keystore                              
  2  from keystore identified by "HiddenPassword2233";

keystore altered.

SQL> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS			       WALLET_TYPE	    WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/cfg/oracle/cdba03/tde/
OPEN_NO_MASTER_KEY	       AUTOLOGIN	    SINGLE    NONE     UNDEFINED
	 1

FILE

OPEN_NO_MASTER_KEY	       AUTOLOGIN	    SINGLE    UNITED   UNDEFINED
	 2

FILE

OPEN_NO_MASTER_KEY	       AUTOLOGIN	    SINGLE    UNITED   UNDEFINED
	 3


SQL> 
```

### Create TDE Master Encryption Key

We can now generate TDE master encryption key on the primary database. 

> [!NOTE]
> Creating a new TDE master encryption key does not encrypt the database objects. But this key is required to start the encryption.

```
SQL> administer key management set key 
  2  using tag 'master_key_01' force keystore
  3  identified by "HiddenPassword2233"
  4  with backup using 'master_key_01_backup01' container=all
  5  /
keystore altered.

SQL> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS			       WALLET_TYPE	    WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/cfg/oracle/cdba03/tde/
OPEN			       PASSWORD 	    SINGLE    NONE     NO
	 1

FILE

OPEN			       AUTOLOGIN	    SINGLE    UNITED   NO
	 2

FILE

OPEN			       PASSWORD 	    SINGLE    UNITED   NO
	 3


SQL> 

```

### Copy TDE Wallet to Standby Database

We have created a TDE wallet with the first master encryption key on the primary database. We can now copy the corresponding files to the standby database. Both primary and standby database have got the same *WALLET_ROOT* directory: */cfg/oracle/cba03*.

```
oracle@bol7rac2a> cd /cfg/oracle/cdba03/

oracle@bol7rac2a> ls
tde

oracle@bol7rac2a> ls tde/
cwallet.sso  ewallet_2025072819583837_master_key_01_backup01.p12  ewallet.p12

oracle@bol7rac2a> pwd
/cfg/oracle/cdba03

oracle@bol7rac2a> scp -r * rkol7rac2a:$PWD
cwallet.sso                                                       100% 5878     6.7MB/s   00:00    
ewallet.p12                                                       100% 5833    10.2MB/s   00:00    
ewallet_2025072819583837_master_key_01_backup01.p12               100% 2553     5.2MB/s   00:00    
oracle@bol7rac2a> 
```

### Check TDE Wallet on Standby Database

Check the status of TDE wallet:
```
SQL> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS			       WALLET_TYPE	    WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/cfg/oracle/cdba03/tde/
OPEN			       AUTOLOGIN	    SINGLE    NONE     NO
	 1


SQL> 
```

It looks different. Let's check the status of PDBs:
```
SQL> show pdbs

    CON_ID CON_NAME			  OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
	 2 PDB$SEED			      MOUNTED
	 3 CDBA03_PDB1			  MOUNTED
SQL> 
```

The PDBs are only mounted. This standby database was started with SRVCTL STOP DATABASE using the start option *mount*. We can start this database using the start option *read only*:
```
oracle@rkol7rac2a> srvctl stop database -db cdba03_dc2
oracle@rkol7rac2a> srvctl start database -db cdba03_dc2 -startoption "read only"
```

The new status looks similar to the primary database:
```
SQL> select * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS			       WALLET_TYPE	    WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/cfg/oracle/cdba03/tde/
OPEN			       AUTOLOGIN	    SINGLE    NONE     NO
	 1

FILE

OPEN			       AUTOLOGIN	    SINGLE    UNITED   NO
	 2

FILE

OPEN			       AUTOLOGIN	    SINGLE    UNITED   NO
	 3

```