+++
title = 'ARUTIL List Command'
date = 2025-06-05T08:15:16+02:00
draft = false
+++

## List Hosts

Use `arutil list hosts` command to fetch the configuration details about host names and network addresses. This command provides the following information:
- Public host names
- Cluster VIPs 
- Cluster networks
- SCANs.

Example Output:
```
oracle@bol7rac1a> ./arutil_ol7 list hosts
Cluster Nodes:
 - host name: bol7rac1a
   => network number: 1  VIP name: bol7rac1av
      IPv4: 192.168.5.32
   => network number: 2  VIP name: bol7rac1av8
      IPv4: 192.168.9.32

 - host name: bol7rac1b
   => network number: 1  VIP name: bol7rac1bv
      IPv4: 192.168.5.34
   => network number: 2  VIP name: bol7rac1bv8
      IPv4: 192.168.9.34

Cluster Networks:
 - network number: 1
    IPv4: 192.168.4.0/255.255.252.0/eth0, static
 - network number: 2
    IPv4: 192.168.8.0/255.255.252.0/eth2, static

Cluster SCANs:
 - SCAN Name: bscan1
    Network Number: 1
    IPv4: 192.168.4.0/255.255.252.0/eth0, static
    SCAN Address: 192.168.5.250
    SCAN Address: 192.168.5.249
    SCAN Address: 192.168.5.248

 - SCAN Name: bscan1-dg
    Network Number: 2
    IPv4: 192.168.8.0/255.255.252.0/eth2, static
    SCAN Address: 192.168.9.249
    SCAN Address: 192.168.9.250
    SCAN Address: 192.168.9.248

oracle@bol7rac1a> 
```

## List Listeners

The command `arutil list listeners` displays the configuration and current status of following cluster listeners:
- Database listeners
- SCAN listeners

Example Output:
```
oracle@bol7rac1a> ./arutil_ol7 list listeners
Listeners:
 - Listener Name: DG_LISTENER
     Network Number: 2
     EndPoints: TCP:1522
     Type: Database Listener
     Owner: grid
     Home: <CRS home>
     ORACLE_HOME: /u01/grid_home
     Status:
      - Listener DG_LISTENER is enabled
      - Listener DG_LISTENER is running on node(s): bol7rac1a,bol7rac1b
 - Listener Name: LISTENER
     Network Number: 1
     EndPoints: TCP:1521
     Type: Database Listener
     Owner: grid
     Home: <CRS home>
     ORACLE_HOME: /u01/grid_home
     Status:
      - Listener LISTENER is enabled
      - Listener LISTENER is running on node(s): bol7rac1a,bol7rac1b

SCAN Listeners:
 - Network Number: 1
    EndPoints: TCP:1521
    Names:
      - LISTENER_SCAN1
      - LISTENER_SCAN2
      - LISTENER_SCAN3
    Status:
      - SCAN Listener LISTENER_SCAN1 is enabled
      - SCAN listener LISTENER_SCAN1 is running on node bol7rac1a
      - SCAN Listener LISTENER_SCAN2 is enabled
      - SCAN listener LISTENER_SCAN2 is running on node bol7rac1a
      - SCAN Listener LISTENER_SCAN3 is enabled
      - SCAN listener LISTENER_SCAN3 is running on node bol7rac1a
 - Network Number: 2
    EndPoints: TCP:1522
    Names:
      - DG_LISTENER_SCAN1_NET2
      - DG_LISTENER_SCAN2_NET2
      - DG_LISTENER_SCAN3_NET2
    Status:
      - SCAN Listener DG_LISTENER_SCAN1_NET2 is enabled
      - SCAN listener DG_LISTENER_SCAN1_NET2 is running on node bol7rac1b
      - SCAN Listener DG_LISTENER_SCAN2_NET2 is enabled
      - SCAN listener DG_LISTENER_SCAN2_NET2 is running on node bol7rac1a
      - SCAN Listener DG_LISTENER_SCAN3_NET2 is enabled
      - SCAN listener DG_LISTENER_SCAN3_NET2 is running on node bol7rac1a
oracle@bol7rac1a> 
```