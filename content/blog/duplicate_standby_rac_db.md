+++
title = 'How To Create Standby RAC Database With RMAN DUPLICATE'
date = 2025-05-29T18:13:54+02:00
draft = true
summary = """
There are different RMAN DUPLICATE methods to create a standby RAC database. This document describes active database duplication using either image copies or backup sets.
"""
tags = ["rac", "standby", "rman", "data guard", "duplicate"]
+++


The example commands use the following configuration.

There are two RAC clusters consisting of the nodes *bol7rac1a*/*bol7rac1b* and *rkol7rac1a*/*rkol7rac1b*. The OS user "grid" was used to install the Grid Infrastructure. There are two ASM disk groups DATA and FRA on each cluster.

The RAC databases were installed using the OS user "oracle". The primary RAC database *cdbrac* runs on the servers *bol7rac1* and *bol7rac2* and uses ASM. The primary database parameters:
- DB_NAME: cdbrac
- DB_UNIQUE_NAME: cdbrac_db1
- DB_DOMAIN: world

The standby database should run on the servers *rkol7rac1* and *rkol7rac2*. The standby database parameters:
- DB_NAME: cdbrac
- DB_UNIQUE_NAME: cdbrac_db2
- DB_DOMAIN: world

