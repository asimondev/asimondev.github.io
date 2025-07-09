+++
title = 'Using RedoRoutes Property in Data Guard Broker'
date = 2025-07-09T20:10:59+02:00
draft = false
summary = """
Data Guard Broker provides a default redo transport configuration for all primary and standby databases. You can use the RedoRoutes property to create a specific redo transport topology.
"""
tags = ["rac", "standby", "data guard"]
+++

In default Data Guard configuration with one primary database and one standby database the redo transport is clear. But do you want to send all redos for 2 or more standby databases to all standby databases? Sometimes you would like to send the redos to only one standby database. This standby should forward the redos to another database. Of course, it should work even if the first forwarding database is down.

## Example configuration

For this example I am going to use the primary database CDBA02_DC1. There are 2 standby databases CDBA02_DC2 and CDBA02_DC3. These are all RAC database on ASM using Oracle Release 19.26.

Here is the current Data Guard Broker configuration:
```
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 
    cdba02_dc3 - Physical standby database 

Fast-Start Failover:  Disabled
```

Let's assume we don't use the CDBA02_DC3 as a primary database. This should be always a standby database. This database should get redos either from CDBA02_DC1 or from CDBA02_DC2 databases.

I will use the script *unset_routes.dg* to clear the RedoRoutes property.
```
racle@bol7rac2a> cat unset_routes.dg
edit database cdba02_dc1 set property RedoRoutes='';
edit database cdba02_dc2 set property RedoRoutes='';
edit database cdba02_dc3 set property RedoRoutes='';
```

You can find the documentation to this Data Guard Broker Property in [Oracle Data Guard Broker](https://docs.oracle.com/en/database/oracle/oracle-database/19/dgbkr/oracle-data-guard-broker-properties.html#GUID-FB1A96ED-5E6D-43FE-BFC4-6A808E589410).

## CDBA02_DC1 Is Primary Database

Rules description:
- Send redos from the current primary database CDBA02_DC1 to the standby CDBA02_DC2. Only if this database is not available, send the redos directly to the CDBA02_DC3.
- If the primary database is CDBA02_DC1, then the database CDBA02_DC2 should forward redos to the CDBA02_DC3.

```
oracle@bol7rac2a> cat set_routes00.dg
edit database cdba02_dc1 set property RedoRoutes='(local: (cdba02_dc2 priority=1, cdba02_dc3 priority=2))';
edit database cdba02_dc2 set property RedoRoutes='(cdba02_dc1: cdba02_dc3)';
oracle@bol7rac2a> 

DGMGRL> @set_routes00.dg
Warning: ORA-16677: Standby database has the same or higher priority than other members specified in the RedoRoutes group.

Property "redoroutes" updated
Property "redoroutes" updated
DGMGRL> 
```
The Data Guard Broker checks the rules one after another. If we change the order of rules in the script, the error will be fixed.

```
oracle@bol7rac2a> cat set_routes01.dg
edit database cdba02_dc2 set property RedoRoutes='(cdba02_dc1: cdba02_dc3)';
edit database cdba02_dc1 set property RedoRoutes='(local: (cdba02_dc2 priority=1, cdba02_dc3 priority=2))';

DGMGRL> @set_routes01.dg
Property "redoroutes" updated
Property "redoroutes" updated
DGMGRL> 
```

The output of *show configuration* looks different now:

```
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc1 - Primary database
    cdba02_dc2 - Physical standby database 
      cdba02_dc3 - Physical standby database (receiving archived redo)
```

If the *forwarder* database CDBA02_DC2 will be done, the primary database CDBA02_DC1 will send the redos directly to the CDBA02_DC3. If the CDBA02_DC2 is available again, the configuration will be reset.

## Switchover to CDBA02_DC2

Now we would like to switchover to CDBA02_DC2. 

Here is the configuration after the switchover
```
DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc2 - Primary database
    cdba02_dc1 - Physical standby database 
    cdba02_dc3 - Physical standby database 

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 68 seconds ago)
```

Apparently the current rules have no impact:
```
DGMGRL> host
oracle@bol7rac2a> cat show_routes.dg
show database cdba02_dc1 RedoRoutes;
show database cdba02_dc2 RedoRoutes;
show database cdba02_dc3 RedoRoutes;

DGMGRL> @show_routes.dg
  RedoRoutes = '(local: (cdba02_dc2 priority=1, cdba02_dc3 priority=2))'
  RedoRoutes = '(cdba02_dc1: cdba02_dc3)'
  RedoRoutes = ''
DGMGRL> 
```

The both rules do not work, because the current primary database is CDBA02_DC2. This database is not specified as a source database in the rules. So we have to set new rules.

```
oracle@bol7rac2a> cat set_routes02.dg 
edit database cdba02_dc1 set property RedoRoutes='(cdba02_dc2: cdba02_dc3)';
edit database cdba02_dc2 set property RedoRoutes='(local: (cdba02_dc1 priority=1, cdba02_dc3 priority=2))';
```

Now we can clear the old routes and set the new ones.
```
DGMGRL> @unset_routes.dg
Property "redoroutes" updated
Property "redoroutes" updated
Property "redoroutes" updated
DGMGRL>  
DGMGRL> @set_routes02.dg
Property "redoroutes" updated
Property "redoroutes" updated
DGMGRL> 

DGMGRL> show configuration;

Configuration - cdba02_dg

  Protection Mode: MaxPerformance
  Members:
  cdba02_dc2 - Primary database
    cdba02_dc1 - Physical standby database 
      cdba02_dc3 - Physical standby database (receiving archived redo)

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 43 seconds ago)
```

Apparently the usage of RedoRoutes at least with 2 standby databases is not difficult. Just don't forget to update your RedoRoutes property after each switchover.
