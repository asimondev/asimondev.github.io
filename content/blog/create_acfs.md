+++
title = 'How To Create ACFS Mount Point'
date = 2025-07-27T16:19:42+02:00
draft = false
summary = """
Oracle Clusterware manages ASM for Oracle RAC databases. But sometimes you need a cluster file system to store database files or other files. ACFS is a cluster file system. It's based on ASM and is available on Oracle Clusterware. This blog shows how to create a cluster file system mount point.
"""
tags = ["rac", "asm", "acfs"]
+++

## Creating ACFS Mount Point

### Configuration Details

The example commands use the following configuration.

Oracle Clusterware runs on the nodes *rkol7rac2a* and *rkol7rac2b*. There are ASM disk groups DATA, FRA and FS. We want to create a new cluster file system on the ASM disk group FS.

## Create ASM Volume

ACFS runs on ASM in Oracle Clusterware. Every ACFS is located in Oracle ASM Dynamic Volume. We use *asmcmd* to create a new volume in ASM. The minimum volume size is 512MB. Le't create a new 1GB large volume in ASM disk group *FS*. We will use this volume for configuration files and TDE wallets.

The following commands are executed as *grid* OS user:

Check the size of *FS* ASM disk group
```
grid@rkol7rac2a> asmcmd    
ASMCMD> lsdg FS
State    Type    Rebal  Sector  Logical_Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  EXTERN  N         512             512   4096  1048576     20478    19343                0           19343              0             N  FS/
```

There is enough free space to create a new ASM volume:
```
ASMCMD> volcreate -G FS -s 1G cfg
```

## Create ACFS

Let's check the status of the ASM disk volume *cfg*:
```
ASMCMD> volinfo -G FS cfg
Diskgroup Name: FS

	 Volume Name: CFG
	 Volume Device: /dev/asm/cfg-459
	 State: ENABLED
	 Size (MB): 1024
	 Resize Unit (MB): 64
	 Redundancy: UNPROT
	 Stripe Columns: 8
	 Stripe Width (K): 1024
	 Usage: 
	 Mountpath: 
```

In the output we can see the device path for this volume: */dev/asm/cfg-459*. We use this device to create a new cluster file system (ACFS) on it.

As *root* OS user:
```
root@rkol7rac2a ~]# mkfs -t acfs -n cfgfs /dev/asm/cfg-459
mkfs.acfs: version                   = 19.0.0.0.0
mkfs.acfs: on-disk version           = 46.0
mkfs.acfs: volume                    = /dev/asm/cfg-459
mkfs.acfs: volume size               = 1073741824  (   1.00 GB )
mkfs.acfs: Format complete.
[root@rkol7rac2a ~]# 
```

The *-n cfgfs* options specifies *cfgfs* name for our new file system.

## Add ACFS Mount Point

ACFS is created. Now we have to add mount points on each node. As *root* OS user create on both nodes a new directory */cfg*:

```
[root@rkol7rac2a ~]# mkdir /cfg
[root@rkol7rac2a ~]# 

[root@rkol7rac2b ~]# mkdir /cfg
[root@rkol7rac2b ~]# 
```

Now we can register this mount point to ACFS. 

As *root* OS user:
```
[root@rkol7rac2a ~]# /sbin/acfsutil registry -a /dev/asm/cfg-459 /cfg
acfsutil registry: mount point /cfg successfully added to Oracle Registry
```

We can check the ACFS property of this mount point with the following command as *root* OS user:
```
[root@rkol7rac2a ~]# /sbin/acfsutil info fs /cfg
/cfg
    ACFS Version: 19.0.0.0.0
    on-disk version:       49.0
    compatible.advm:       19.0.0.0.0
    ACFS compatibility:    19.0.0.0.0
    flags:        MountPoint,Available,KiloSnap
    creation time:   Sun Jul 27 16:44:43 2025
    mount time:      Sun Jul 27 16:51:42 2025
    mount sequence number: 1
    number of nodes:       2
    allocation unit:       4096
    metadata block size:   4096
    volumes:      1
    total size:   1073741824  (   1.00 GB )
    total free:   496132096  ( 473.15 MB )
    file entry table allocation: 393216
    primary volume: /dev/asm/cfg-459
        label:                 cfgfs
        state:                 Available
        major, minor:          250, 235010
        logical sector size:   512
        size:                  1073741824  (   1.00 GB )
        free:                  496132096  ( 473.15 MB )
        metadata read I/O count:         2096
        metadata write I/O count:        38
        total metadata bytes read:       8806400  (   8.40 MB )
        total metadata bytes written:    540672  ( 528.00 KB )
        ADVM diskgroup:        FS
        ADVM resize increment: 67108864
        ADVM redundancy:       unprotected
        ADVM stripe columns:   8
        ADVM stripe width:     1048576
    number of snapshots:  0
    snapshot space usage: 0  ( 0.00 )
    replication status: DISABLED
    compression status: DISABLED
[root@rkol7rac2a ~]# 
```

## ACFS Test

Let's test this cluster file system as *oracle* OS user. At first, we have to create a new directory in this mount point and give *oracle* OS user read and write permissions on it.

As *root* OS user on the first node:
```
[root@rkol7rac2a ~]# mkdir /cfg/oracle
[root@rkol7rac2a ~]# chown oracle:oinstall /cfg/oracle
[root@rkol7rac2a ~]# ls -ld /cfg/oracle
drwxr-xr-x 2 oracle oinstall 20480 27. Jul 16:56 /cfg/oracle
[root@rkol7rac2a ~]# 
```

As *root* OS user on the second node:
```
[root@rkol7rac2b ~]# ls -ld /cfg/oracle
drwxr-xr-x 2 oracle oinstall 20480 27. Jul 16:56 /cfg/oracle
[root@rkol7rac2b ~]# 
```

We have created a new directory on the first node. The second node is aware of this directory as well. Now we want to create a new file as *oracle* OS user in this directory:

First node:
```
oracle@rkol7rac2a> echo "ACFS test"  > /cfg/oracle/test.out
oracle@rkol7rac2a> 

```

Second node:
```
oracle@rkol7rac2b> cat /cfg/oracle/test.out 
ACFS test
oracle@rkol7rac2b> 
```

ACFS is ready to use on both nodes.

## Resize ACFS

### Manual Resize

If there is enough space in the ASM disk group, you can resize the ACFS mount point. 

We want to change the size of this file system:
```
[root@rkol7rac2a ~]# /sbin/acfsutil info fs /cfg
/cfg
    ACFS Version: 19.0.0.0.0
...
    allocation unit:       4096
    metadata block size:   4096
    volumes:      1
    total size:   1073741824  (   1.00 GB )
    total free:   429023232  ( 409.15 MB )
...
```

We can increase the size by 128MB:

```
[root@rkol7rac2a ~]# /sbin/acfsutil size +128M /cfg
acfsutil size: new file system size: 1207959552 (1152MB)

[root@rkol7rac2a ~]# /sbin/acfsutil info fs /cfg
/cfg
    ACFS Version: 19.0.0.0.0
...
    allocation unit:       4096
    metadata block size:   4096
    volumes:      1
    total size:   1207959552  (   1.12 GB )
    total free:   562974720  ( 536.89 MB )
...
```

### Automatic Resize

You can use *-a IncrementSize* option to automatically resize the ACFS by the specified increment. In such a case you should use *-x MaximumSize* option as well to set the maximum size of this file system.

Enable automatic resize:
```
[root@rkol7rac2a ~]# /sbin/acfsutil size -a 64M -x 2048M /cfg
acfsutil size: ACFS-03641: auto-resize increment below minimum 1G
```

Apparently we should increase the increment from 128MB to 1GB:
```
root@rkol7rac2a ~]# /sbin/acfsutil size -a 1G -x 2048M /cfg
acfsutil size: ACFS-03642: successfully updated auto-resize settings
[root@rkol7rac2a ~]# 
```

Let's check automatic resize properties of this file system:
```
[root@rkol7rac2a ~]# /sbin/acfsutil info fs /cfg
/cfg
    ACFS Version: 19.0.0.0.0
...
    total size:   2147483648  (   2.00 GB )
    total free:   1500635136  (   1.40 GB )
    auto-resize increment:   1073741824  (   1.00 GB )
    auto-resize maximum:     2147483648  (   2.00 GB )
...
```

We don't need automatic resize for this file system. So we disable this now:

```
[root@rkol7rac2a ~]# /sbin/acfsutil size -a 0 /cfg
acfsutil size: ACFS-03642: successfully updated auto-resize settings
[root@rkol7rac2a ~]# 

[root@rkol7rac2a ~]# /sbin/acfsutil info fs /cfg
/cfg
    ACFS Version: 19.0.0.0.0
...
    total size:   2147483648  (   2.00 GB )
    total free:   1500635136  (   1.40 GB )
...
```