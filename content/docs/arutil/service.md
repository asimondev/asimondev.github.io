+++
title = 'ARUTIL Service Command'
date = 2025-06-05T13:17:44+02:00
draft = true
+++

[Add Services](#add-database-services)  
[Compare Services](#compare-database-services)  
[List Services](#list-database-services)  
[Migrate Services](#migrate-database-services)  
[Remove Services](#remove-database-services)  

## Add Database Services

Sometimes you would like to generate *SRVCTL ADD SERVICE* commands based on the output of the existing database. 

> [!TIP]
> At the moment only essential service parameters are supported.

Example Output:
```
oracle@bol7rac1a> ./arutil service add -d cdba01
if srvctl add service -db cdba01 -service cdba01_pdb1_srv01 -pdb cdba01_pdb1 -preferred cdba011,cdba012 -role "PRIMARY" ; then
  echo "Added service cdba01_pdb1_srv01"
else
  echo "Error adding service cdba01_pdb1_srv01"
fi

if srvctl add service -db cdba01 -service cdba01_pdb1_srv02 -pdb cdba01_pdb1 -preferred cdba011 -available cdba012 -role "PRIMARY" ; then
  echo "Added service cdba01_pdb1_srv02"
else
  echo "Error adding service cdba01_pdb1_srv02"
fi


===> 2 services found.
```

If you prefer to get only *SRVCTL ADD SERVICE* commands, you should specify the option **--command-only** / **-c**.

Example Output:
```
oracle@bol7rac1a> ./arutil service add -d cdba01 -c
srvctl add service -db cdba01 -service cdba01_pdb1_srv01 -pdb cdba01_pdb1 -preferred cdba011,cdba012 -role "PRIMARY"
srvctl add service -db cdba01 -service cdba01_pdb1_srv02 -pdb cdba01_pdb1 -preferred cdba011 -available cdba012 -role "PRIMARY"

===> 2 services found.
oracle@bol7rac1a> 
```

## Compare Database Services

In some cases you would like to compare database services between two databases. The possible reasons are:
- Data Guard configurations (primary and standby databases)
- Similar configuration between two database like production and integration environments

You can use *ARUTIL* to compare the output of *SRVCTL CONFIG SERVICE*. You have two options:
- Provide two configuration files generated with *SRVCTL CONFIG SERVICE*
- Provide one such a configuration file and use **--database** / **-d** option to query the service configuration of the current database.

Generate the configuration details for the first database:  
```
oracle@bol7rac1a> srvctl config service -db cdba02 > /home/oracle/tmp/srvctl_config_service_cdba02.out
oracle@bol7rac1a> 
```

Compare the generated configuration file with the current database:
```
oracle@bol7rac1a> ./arutil_ol7 service compare -d cdba01 /home/oracle/tmp/srvctl_config_service_cdba02.out 
Comparing cdba01 services to /home/oracle/tmp/srvctl_config_service_cdba02.out

Comparing RAC Services...
 - Different number of services: 2 vs 3
 - Service cdba01_pdb1_srv01a not found in 1. file
 - Name: cdba01_pdb1_srv01a
    Server Pool                   : 
    Cardinality                   : 2
    Role                          : PRIMARY
    Management Policy             : AUTOMATIC
    DTP Transaction               : false
    AQ HA Notifications           : false
    Global                        : false
    Commit Outcome                : false
    Failover Type                 : 
    Failover Method               : 
    Failover Retries              : 
    Failover Delay                : 
    Failover Restore              : NONE
    Connection Load Balancing Goal: LONG
    Runtime Load Balancing Goal   : NONE
    TAF Policy Specification      : NONE
    Edition                       : 
    PDB                           : cdba01_pdb1
    Hub Service                   : 
    Maximum Lag Time              : ANY
    SQL Translation Profile       : 
    Retention                     : 86400 seconds
    Replay Initiation Time        : 300 seconds
    Drain Timout                  : 
    Stop Option                   : 
    Session State Consistency     : DYNAMIC
    GSM Flags                     : 0
    Status                        : enabled
    Preferred instances           : cdba011,cdba012
    Available instances           : 
    CSS Critical                  : no
    Java Usage                    : false

 - Service: cdba01_pdb1_srv01
   - Role:
   >> First : PRIMARY
   >> Second: PRIMARY,PHYSICAL_STANDBY


==> RAC Services are different.

oracle@bol7rac1a> 
```

Sometimes you want to ignore some differences in configuration details. You can use "--config-json"/"-j" parameter to specify a JSON configuration file with your adjustments.

Here is the default *ARUTIL* configuration file with all default options:
```
{
"ignore_instance_name" : false,
"ignore_instance_count" : false,
"ignore_java_usage" : false
}
```
You could provide your own JSON configuration with the modified flags:
 - "ignore_instance_name" : **true** => Only the number of instances in preferred / available parameters must be equal
 - "ignore_instance_count" : **true** => At least one instance should be available in preferred / available parameters
 - "ignore_java_usage" : **true** => Ignore the difference for the parameter "Service uses Java"

For instance, we would like to compare two databases with the similar services. But the instance names are different between these databases.

```
oracle@bol7rac1a> ./arutil_ol7 service compare ~/tmp/srvctl_config_service_01.out ~/tmp/srvctl_config_service_02.out
Comparing /home/oracle/tmp/srvctl_config_service_01.out to /home/oracle/tmp/srvctl_config_service_02.out

Comparing RAC Services...
 - Service: cdba01_pdb1_srv01
   - Preferred instances:
   >> First : cdba011,cdba012
   >> Second: cdba021,cdba022

 - Service: cdba01_pdb1_srv02
   - Preferred instances:
   >> First : cdba011
   >> Second: cdba021
   - Available instances:
   >> First : cdba012
   >> Second: cdba022


==> RAC Services are different.
```

We could create a configuration file *~/tmp/config.json* to ignore the differences in the instance names. But the number of instances must be the same!

```
oracle@bol7rac1a> cat ~/tmp/config.json 
{
"ignore_instance_name" : true
}
oracle@bol7rac1a> 

oracle@bol7rac1a> ./arutil_ol7 service compare ~/tmp/srvctl_config_service_01.out ~/tmp/srvctl_config_service_02.out -j ~/tmp/config.json
Comparing /home/oracle/tmp/srvctl_config_service_01.out to /home/oracle/tmp/srvctl_config_service_02.out

Comparing RAC Services...



==> RAC Services are different.

oracle@bol7rac1a> 
```

We still have got a result message, that the RAC services are different. But the service comparison output is empty.

## List Database Services

Use `arutil service list` command with **--database** / **-d** option to query database service details. *ARUTIL* would present the results of the following RAC commands:
- *RAC_HOME*/bin/srvctl config service -db*
- *RAC_HOME*/bin/srvctl status service -db*

Example Output:
```
oracle@bol7rac1a> ./arutil service list -d cdba01
=== Database Service Configuration ===

Service name: cdba01_pdb1_srv01
Server pool: 
Cardinality: 2
Service role: PRIMARY
Management policy: AUTOMATIC
DTP transaction: false
AQ HA notifications: false
Global: false
Commit Outcome: false
Failover type: 
Failover method: 
Failover retries: 
Failover delay: 
Failover restore: NONE
Connection Load Balancing Goal: LONG
Runtime Load Balancing Goal: NONE
TAF policy specification: NONE
Edition: 
Pluggable database name: cdba01_pdb1
Hub service: 
Maximum lag time: ANY
SQL Translation Profile: 
Retention: 86400 seconds
Replay Initiation Time: 300 seconds
Drain timeout: 
Stop option: 
Session State Consistency: DYNAMIC
GSM Flags: 0
Service is enabled
Preferred instances: cdba011,cdba012
Available instances: 
CSS critical: no
Service uses Java: false
Service name: cdba01_pdb1_srv02
...
Preferred instances: cdba011
Available instances: cdba012
CSS critical: no
Service uses Java: false


=== Database Service Status ===

Service cdba01_pdb1_srv01 is running on instance(s) cdba011,cdba012
Service cdba01_pdb1_srv02 is running on instance(s) cdba012


oracle@bol7rac1a> 
```

## Migrate Database Services

*ARUTIL* can help to generate *SRVCTL ADD SERVICE* commands for database migration. In such a case you need to provide the following configuration details:
- Database service configuration. You could save the output of *SRVCTL CONFIG SERVICE* into a file and use it with *--old-config* / *-o* parameter. On the other side with the *--database* / *-d* parameter you could let *ARUTIL* fetch the service configuration on its own.
- Target configuration using *--new-config* / *-n* parameter. 

The target configuration should be a JSON configuration file with the following possible mapping parameters:

 - `old_database`: old database unique name
 - `new_database`: new database unique name
 - `pdb`: new PDB name
 - `add_preferred`: additional preferred instances
 - `add_available`: additional available instances

> [!TIP]
> At the moment only essential service parameters are supported.

### Examples

We would like to generate a *SRVCTL ADD SERVICE* commands using the current database. The new database name is *cdba02*. 

We use **--command** / **-c** option to generate only the *SRVCTL* commands.

```
oracle@bol7rac1a> cat ~/tmp/migrate.json 
{
 "old_database" : "cdba01",
 "new_database" : "cdba02"
}

oracle@bol7rac1a> ./arutil_ol7 service migrate -d cdba01 --new-config ~/tmp/migrate.json -c
srvctl add service -db cdba02 -service cdba01_pdb1_srv01 -pdb cdba01_pdb1 -preferred cdba011,cdba012 -role "PRIMARY"
srvctl add service -db cdba02 -service cdba01_pdb1_srv02 -pdb cdba01_pdb1 -preferred cdba011 -available cdba012 -role "PRIMARY"
oracle@bol7rac1a> 
```

In the next step we would like to rename the PDB from *cdba01_pdb1* to *mypdb*.

```
oracle@bol7rac1a> cat ~/tmp/migrate.json 
{
 "old_database" : "cdba01",
 "new_database" : "cdba02",
 "pdb": "mypdb"
}

oracle@bol7rac1a> ./arutil_ol7 service migrate -d cdba01 --new-config ~/tmp/migrate.json -c
srvctl add service -db cdba02 -service cdba01_pdb1_srv01  -pdb mypdb -preferred cdba011,cdba012 -role "PRIMARY"
srvctl add service -db cdba02 -service cdba01_pdb1_srv02  -pdb mypdb -preferred cdba011 -available cdba012 -role "PRIMARY"
oracle@bol7rac1a> 
```

The new database has one additional instance. We would like to add this instance to all available instances.

```
oracle@bol7rac1a> cat ~/tmp/migrate.json 
{
 "old_database" : "cdba01",
 "new_database" : "cdba02",
 "pdb": "mypdb",
 "add_available" : "cdba023"
}
oracle@bol7rac1a> 

oracle@bol7rac1a> ./arutil_ol7 service migrate -d cdba01 --new-config ~/tmp/migrate.json -c
srvctl add service -db cdba02 -service cdba01_pdb1_srv01  -pdb mypdb -preferred cdba011,cdba012 -available cdba023 -role "PRIMARY"
srvctl add service -db cdba02 -service cdba01_pdb1_srv02  -pdb mypdb -preferred cdba011 -available cdba012,cdba023 -role "PRIMARY"
oracle@bol7rac1a> 
```


## Remove Database Services

Sometimes you have to create a script to stop and remove all database services. You could use the existing database to generate such a script. 

> [!IMPORTANT]
> *ARUTIL* uses the specified database to fetch database services configuration. No service will be stopped or removed on this database!


Example Output:
```
oracle@bol7rac1a> ./arutil service remove -d cdba01 
if srvctl stop service -db cdba01 -service cdba01_pdb1_srv01 -stopoption immediate -force ; then
  echo "Stopped service cdba01_pdb1_srv01"
else
  echo "Error stopping service cdba01_pdb1_srv01"
fi

if srvctl remove service -db cdba01 -service cdba01_pdb1_srv01  ; then
  echo "Removed service cdba01_pdb1_srv01"
else
  echo "Error removing service cdba01_pdb1_srv01"
fi

if srvctl stop service -db cdba01 -service cdba01_pdb1_srv02 -stopoption immediate -force ; then
  echo "Stopped service cdba01_pdb1_srv02"
else
  echo "Error stopping service cdba01_pdb1_srv02"
fi

if srvctl remove service -db cdba01 -service cdba01_pdb1_srv02  ; then
  echo "Removed service cdba01_pdb1_srv02"
else
  echo "Error removing service cdba01_pdb1_srv02"
fi

oracle@bol7rac1a> 
```

If you prefer to get only *SRVCTL STOP SERVICE* / *SRVCTL REMOVE SERVICE* commands, you should specify the option **--command-only** / **-c**.

Example Output:
```
oracle@bol7rac1a> ./arutil service remove -d cdba01 -c
srvctl stop service -db cdba01 -service cdba01_pdb1_srv01 -stopoption immediate -force
srvctl remove service -db cdba01 -service cdba01_pdb1_srv01 
srvctl stop service -db cdba01 -service cdba01_pdb1_srv02 -stopoption immediate -force
srvctl remove service -db cdba01 -service cdba01_pdb1_srv02 
oracle@bol7rac1a> 
```