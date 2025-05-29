+++
title = 'Delete Online Redo Logs'
date = 2025-05-29T12:09:07+02:00
draft = false
summary = """
Sometimes you have to add larger online redo log files and delete the smaller ones. The attached script helps to delete the existing smaller log files.
"""
tags = ["standby", "redo", "data guard"]
+++

Sometimes you need to increase the size of the online redo log files. In such a case you would make these steps:
1. Use **ALTER DATABASE ADD LOGFILE** to add new larger log files.
2. Use **ALTER DATABASE DROP LOGFILE** to drop the existing smaller log files.

As you know the online log files could be in status **ACTIVE**, **CURRENT**, **INACTIVE** etc. For example,  it's not possible to delete an online redo log file, if it's in status **CURRENT**. You have to switch and archive such a redo log file before you can delete it. In RAC the number of online log files are even higher because of additional number of redo threads.

## Download SQL Script

The attached PL/SQL script will delete existing smaller log files in all threads. You should save this script in a file and run it from your SQL*Plus session as a *DBA* user.

```
-- Created by Andrej Simon, Oracle Germany
--

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

## Check Online Redo Log Files

Let's check the existing online redo log files after adding some larger files with **ALTER DATABASE ADD LOGFILE**.

```
SQL> select thread# thr, round(bytes/1024/1024, 0) mb, group# grp from v$log order by thr, mb, grp;

       THR	   MB	     GRP
---------- ---------- ----------
	 1	  200	       1
	 1	  200	       2
	 1	  256	       5
	 1	  256	       6
	 2	  200	       3
	 2	  200	       4
	 2	  256	       7
	 2	  256	       8

8 rows selected.
```

This is a RAC database with 2 redo threads. In each thread we have 2 smaller log files with 200 MB (the "old" ones) and 2 larger log files with 256 MB (the "new" ones). We need to delete the 4 smaller log files.

## Deleting Log Files

Use copy and paste to transfer a copy of the above script into the new file *delete_logs.sql*. You can run this script from SQL*Plus using a *SYS* or other DBA user.

```
oracle@bol7rac1a> sqlplus / as sysdba
...

SQL> @delete_logs
SQL> set numwidth 20
SQL> set serveroutput on
SQL> 
SQL> select thread# thr, bytes, count(*) cnt
  2  from v$log
  3  group by thread#, bytes
  4  order by 1, 2
  5  /

		 THR		    BYTES		   CNT
-------------------- -------------------- --------------------
		   1		209715200		     2
		   1		268435456		     2
		   2		209715200		     2
		   2		268435456		     2

SQL> 
SQL> prompt Please specify the bytes in online log files to delete:
Please specify the bytes in online log files to delete:
SQL> accept my_bytes number
209715200
SQL> 
SQL> declare
  2    l_wrong_size_bytes number := &my_bytes;
  3  
  4    function log_files_count(log_file_size number) return number
  5    as
  6  	 cnt number;
  7    begin
  8  	 select count(*) into cnt from v$log
  9  	 where bytes = log_file_size;
 10  
 11  	 return cnt;
 12    end log_files_count;
 13  
 14    procedure delete_logs(l_wrong_size_bytes number) is
 15  	 cmd varchar2(512) := 'alter database drop logfile group ';
 16  	 cursor c1 is
 17  	   select thread# thr, group# grp from v$log
 18  	   where bytes = l_wrong_size_bytes and
 19  	     status in ('INACTIVE', 'UNUSED');
 20    begin
 21  	 for log_file in c1 loop
 22  	   dbms_output.put_line('Deleting thread ' || log_file.thr ||
 23  	       ' group ' || log_file.grp);
 24  	   execute immediate cmd || log_file.grp;
 25  	 end loop;
 26    end delete_logs;
 27  
 28    procedure switch_logs(l_wrong_size_bytes number) is
 29    begin
 30  	 if log_files_count(l_wrong_size_bytes) = 0 then
 31  	   return;
 32  	 end if;
 33  
 34  	 execute immediate 'alter system archive log current';
 35  	 execute immediate 'alter system checkpoint';
 36  	 dbms_session.sleep(10);
 37    end switch_logs;
 38  begin
 39  
 40    loop
 41  	 exit when log_files_count(l_wrong_size_bytes) = 0;
 42  
 43  	 delete_logs(l_wrong_size_bytes);
 44  
 45  	 switch_logs(l_wrong_size_bytes);
 46    end loop;
 47  
 48  end;
 49  /
old   2:   l_wrong_size_bytes number := &my_bytes;
new   2:   l_wrong_size_bytes number := 	   209715200;
Deleting thread 1 group 1
Deleting thread 2 group 3
Deleting thread 1 group 2
Deleting thread 2 group 4

PL/SQL procedure successfully completed.

SQL> 
SQL> col thr for 999
SQL> col bytes for 999999999999
SQL> col mb for 999999999
SQL> col cnt for 999
SQL> 
SQL> select thread# thr, bytes, round(bytes/1024/1024, 0) mb, count(*) cnt
  2  from v$log
  3  group by thread#, bytes
  4  order by 1, 2
  5  /

 THR	     BYTES	   MB  CNT
---- ------------- ---------- ----
   1	 268435456	  256	 2
   2	 268435456	  256	 2

SQL> 
```


