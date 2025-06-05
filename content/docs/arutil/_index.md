+++
title = "Andrej's RAC Utility"
date = 2025-06-05T09:35:39+02:00
draft = false
+++

The *arutil* GitHub repository contains a command line utility for working with Oracle RAC.

*ARUTIL* does not change any RAC configuration on its own. *ARUTIL* only runs some RAC commands to collect the RAC configuration information. This information will be presented to you for monitoring or making some changes. 

*ARUTIL* could run the following GI/RAC commands:
- CRS_HOME/bin/olsnodes
- CRS_HOME/bin/srvctl config
- CRS_HOME/bin/srvctl status
- RAC_HOME/bin/srvctl config
- RAC_HOME/bin/srvctl status

This utility is developed for DBAs. That's why this command should be executed by DBA's OS user account (*oracle*). The RAC environment (*ORACLE_HOME*, *PATH*, *LD_LIBRARY_PATH*) must be set before starting *ARUTIL*.
