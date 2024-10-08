+++
title = 'Cloning'
date = 2024-09-29T22:07:43+02:00
draft = false
+++

## Database 19c Out-of-Place Patching.

You can use *install_base.sh* script to execute out-of-place patching for the
existing 19c database software. The required steps are:
1. Create a gold image of the existing ORACLE_HOME.
1. Create a new ORACLE_HOME using the gold image from the previous step.
1. Patch the new created ORACLE_HOME.
1. Use the new ORACLE_HOME on this server or create a new gold image and 
use it on another servers.

### Creating a new gold image.

Set ORACLE environment and run the following steps to create a new gold image in 
the directory */u01/oracle/images*.

    cd $ORACLE_HOME
    ./runInstaller -silent -createGoldImage -destinationLocation /u01/oracle/images

The created gold image has the automatically generated file name. Usually would like to 
rename it. In the example below we would rename it to *dbhome01.zip*:
```
cd /u01/oracle/images
mv GoldImageName.zip dbhome01.zip
```

### Create a new ORACLE_HOME using the gold image.

At first you have to prepare a new environment file with required environment variables 
for the Oracle installation. For instance, you can use such example file 
*/home/oracle/inst_dbhome02* to install Oracle database software into the new directory 
*/u01/oracle/dbhome_02* :

```
cat /home/oracle/inst_dbhome02
#!/bin/bash

export ORACLE_BASE=/u01/oracle
export ORACLE_HOME=$ORACLE_BASE/dbhome_02
export TMP=/tmp
export TMPDIR=/tmp
umask 022

chmod 755 ~/inst_dbhome02
```

This command would start the database installation:

`./install_base.sh -e ~/inst_dbhome02 -i /u01/oracle/images/dbhome01.zip`

After the installation you have to run the *root.sh* script as *root* user.

Sometimes you have to use **-g** option to specify another UNIX groups. Don't forget
to use **-r*" option for RAC installations.

### Install Oracle patches into the new ORACLE_HOME.

Now you can set the environment variables for the new ORACLE_HOME and install the
required patches using OPatch utility.

You can use this example script */home/oracle/set_dbhome02* to set your environment variables:
```
cat ~/set_dbhome02
export ORACLE_BASE=/u01/oracle
export ORACLE_HOME=$ORACLE_BASE/dbhome_02
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
export VISUAL="$(which vi)"

chmod 755 ~/set_dbhome02
```

Usually you would at first download and install OPatch utitlity (database patch with the id
*6880880*) in the new ORACLE_HOME. You can find OPatch in *MOS*: [Download OPatch](http://updates.oracle.com/ARULink/PatchDetails/process_form?patch_num=6880880)
```
. ~/set_dbhome02
cd $ORACLE_HOME
unzip -o /PatchToPatches/p6880880_190000_Linux-x86-64.zip
```

#### RAC Database Release Update.

On RAC you have to use the GI release update on each node using *opatchauto* utility 
as *root* user.

As *oracle* user:
```
mkdir GI_RU_Patch_Path_Directory
cd GI_RU_Patch_Path_Directory
unzip Path_to_GI_RU_PATCH
```
As root user:
```
export PATH=/u01/oracle/dbhome_02/OPatch:$PATH
opatchauto apply GI_RU_Patch_Path_Directory/PatchNumber -oh /u01/oracle/dbhome_02
```

#### Single Instance Database Release Update.

As oracle user:
```
mkdir DB_RU_Patch_Path_Directory
cd DB_RU_Patch_Path_Directory
unzip Path_to_DB_RU_PATCH
cd PatchNumber
```

After that you can use *opatch apply" to apply the patches as usual.


### Create a new gold image.

Now you have got a new patched ORACLE_HOME. If you like, you can create a 
new gold image to use it on this server later or on another servers.

### Move database(s) to the new ORACLE_HOME.

Please make sure, that all required files from $ORACLE_HOME/network/admin and
$ORACLE_HOME/dbs are copied to the new ORACLE_HOME. Sometimes you have to modify 
these network files to set the new ORACLE_HOME.

If you use RAC, you have to run **srvctl** to set the new ORACLE_HOME:

`srvctl modify database -db mydb -oraclehome /u01/oracle/dbhome_02`

For a single instance database you have to copy the spfile and password files to
the new *$ORACLE_HOME/dbs* directory.

### How to patch RAC faster?

On RAC you have to install and patch every node. It could be sometimes faster to 
use a gold image for the installation on the first node only (*-l* option)

`./install_base.sh -r -l ...`

You have to specify both **-r** (RAC) and **-l** (local node only) options.

After running *root.sh* you would install all patches on this RAC node only. In the 
next step you can create a new gold image from it. Now you could delete the local
installed ORACLE_HOME and use the created gold image to install on all RAC nodes.

`./install_base.sh -r ...`

The **-r** option (RAC) installs per default on all cluster nodes.

### Examples of out-of-place patching.

- [Single Instance Cloning and Patching]({{< relref "cloning_single_instance.md" >}})
- [RAC Cloning and Patching]({{< relref "cloning_rac.md" >}})
- [RAC Cloning and Patching Using Gold Image]({{< relref "cloning_rac_gold_image.md" >}})

