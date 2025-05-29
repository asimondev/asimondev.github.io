+++
title = 'Delete Online Redo Logs'
date = 2025-05-29T12:09:07+02:00
draft = true
summary = """
Sometimes you have to add larger online redo log files and delete the smaller ones. The attached script helps to delete the existing smaller log files.
"""
tags = ["standby", "redo", "data guard"]
+++

Sometimes you need to increase the size of the online redo log files. In such a case you would make these steps:
1. Use **ALTER DATABASE ADD LOGFILE** to add new larger log files.
2. Use **ALTER DATABASE DROP LOGFILE** to drop the existing smaller log files.

As you know the online log files could be in status **ACTIVE**, **CURRENT**, **INACTIVE** etc. For example,  it's not possible to delete an online redo log file, if it's in status **CURRENT**. You have to switch and archive such a redo log file before you can delete it. In RAC the number of online log files are even higher because of additional number of redo threads.

The attached PL/SQL script will delete existing smaller log files. You should save this script in a file and run it from your SQL*Plus session as a *DBA* user.

```
set echo on verify on
set numwidth 20
set serveroutput on

select thread# thr, bytes, count(*) cnt
from v$log
group by thread#, bytes
order by 1, 2
/

prompt Please specify the bytes in online log files to delete:
accept my_bytes number 

declare
  l_wrong_size_bytes number := &my_bytes;

  function log_files_count(log_file_size number) return number
  as
    cnt number;
  begin
    select count(*) into cnt from v$log 
    where bytes = log_file_size;

    return cnt;
  end log_files_count;

  procedure delete_logs(l_wrong_size_bytes number) is
    cmd varchar2(512) := 'alter database drop logfile group ';
    cursor c1 is 
      select thread# thr, group# grp from v$log 
      where bytes = l_wrong_size_bytes and
        status in ('INACTIVE', 'UNUSED');    
  begin
    for log_file in c1 loop
      dbms_output.put_line('Deleting thread ' || log_file.thr || 
          ' group ' || log_file.grp);
      execute immediate cmd || log_file.grp;
    end loop;
  end delete_logs;

  procedure switch_logs(l_wrong_size_bytes number) is
  begin
    if log_files_count(l_wrong_size_bytes) = 0 then
      return;
    end if;

    execute immediate 'alter system archive log current';
    execute immediate 'alter system checkpoint';
    dbms_session.sleep(10);
  end switch_logs;
begin

  loop
    exit when log_files_count(l_wrong_size_bytes) = 0;

    delete_logs(l_wrong_size_bytes);

    switch_logs(l_wrong_size_bytes);
  end loop;

end;
/

col thr for 999
col bytes for 999999999999
col mb for 999999999
col cnt for 999

select thread# thr, bytes, round(bytes/1024/1024, 0) mb, count(*) cnt
from v$log
group by thread#, bytes
order by 1, 2
/
```