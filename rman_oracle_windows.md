# windows平台下 oracle 之间使用 RMAN 进行数据库恢复操作

## 环境

- 源主机

```
ip：172.42.1.160
os：Windows Server 2008 R2 Standard SP1 64bit
内存：8GB
NLS_LANG：AMERICAN_CHINA.ZHS16GBK
oracle：Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
ORACLE_HOME：D:\app\Administrator\product\11.2.0
备份文件目录：D:\backup
闪回区目录：D:\app\Administrator\flash_recovery_area
数据库文件目录：D:\app\Administrator\oradata\orcl
```

- 目标主机

```
ip：172.42.1.162
os：Windows Server 2008 R2 Standard SP1 64bit
内存：8GB
NLS_LANG：AMERICAN_CHINA.ZHS16GBK
oracle：Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
ORACLE_HOME：D:\app\Administrator\product\11.2.0
备份文件目录：D:\backup
闪回区目录：D:\app\Administrator\flash_recovery_area
数据库文件目录：D:\app\Administrator\oradata\orcl
```

## 概述

使用 Rman 备份源主机数据库全库，在目标主机上使用源主机的备份文件进行 RMAN 恢复，备份和恢复前需清理各自原有的备份文件，以及目标主机闪回区的控制文件。

## 步骤

### 源主机

`cmd`

> sqlplus / as sysdba

`sqlplus`

> shutdown immediate

```
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
```

断开所有客户端连接

`sqlplus`

> startup

```
SQL> startup
ORACLE instance started.

Total System Global Area 3423965184 bytes
Fixed Size                  2180544 bytes
Variable Size            2566916672 bytes
Database Buffers          838860800 bytes
Redo Buffers               16007168 bytes
Database mounted.
Database opened.
```

`cmd`

> lsnrctl stop

```
C:\Users\Administrator>lsnrctl stop

LSNRCTL for 64-bit Windows: Version 11.2.0.1.0 - Production on 25-JUL-2018 17:06:37

Copyright (c) 1991, 2010, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=WIN-P1AAHOLITRS)(PORT=1521)))
The command completed successfully
```

`sqlplus`

> alter system switch logfile;
> alter system switch logfile;
> alter system switch logfile;
> alter system switch logfile;
> alter system switch logfile;
> alter system switch logfile;
> alter system switch logfile;

多次执行日志切换，以确保在线日志写入归档日志

~~进入数据备份位置，删除原有老备份~~


`cmd`

> rman target /

进入 RMAN 命令行

> backup database format 'D:\backup\full_%U.bak';

```
RMAN> backup database format 'D:\backup\full_%U.bak';

Starting backup at 25-JUL-18
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00001 name=D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSTEM01.DBF
input datafile file number=00002 name=D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSAUX01.DBF
input datafile file number=00005 name=D:\APP\ADMINISTRATOR\ORADATA\ORCL\LHCJ_TJ.DBF
input datafile file number=00003 name=D:\APP\ADMINISTRATOR\ORADATA\ORCL\UNDOTBS01.DBF
input datafile file number=00004 name=D:\APP\ADMINISTRATOR\ORADATA\ORCL\USERS01.DBF
channel ORA_DISK_1: starting piece 1 at 25-JUL-18
channel ORA_DISK_1: finished piece 1 at 25-JUL-18
piece handle=D:\BACKUP\FULL_0IT8TCPN_1_1.BAK tag=TAG20180725T172159 comment=NONE

channel ORA_DISK_1: backup set complete, elapsed time: 00:01:56
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current control file in backup set
including current SPFILE in backup set
channel ORA_DISK_1: starting piece 1 at 25-JUL-18
channel ORA_DISK_1: finished piece 1 at 25-JUL-18
piece handle=D:\BACKUP\FULL_0JT8TCTB_1_1.BAK tag=TAG20180725T172159 comment=NONE

channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 25-JUL-18
```

> backup current controlfile format 'D:\backup\con_%U.ctl';

```
RMAN> backup current controlfile format 'D:\backup\con_%U.ctl';

Starting backup at 25-JUL-18
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current control file in backup set
channel ORA_DISK_1: starting piece 1 at 25-JUL-18
channel ORA_DISK_1: finished piece 1 at 25-JUL-18
piece handle=D:\BACKUP\CON_0KT8TD3F_1_1.CTL tag=TAG20180725T172711 comment=NONE
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
Finished backup at 25-JUL-18
```

至此，源主机上生成了 3 个文件，其中一个控制文件，两个数据文件。

```
CON_0KT8TD3F_1_1.CTL
FULL_0IT8TCPN_1_1.BAK
FULL_0JT8TCTB_1_1.BAK
```

将这 3 个文件拷贝到目标主机下同路径的备份目录中。

源主机的 RMAN 可以退出了，接下来是目标主机上的操作。


### 目标主机

删除数据文件目录中的所有文件以及闪回目录下的控制文件

`sqlplus`

> shutdown immediate

```
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
```

`sqlplus`

> startup nomount

```
SQL> startup nomount
ORACLE instance started.

Total System Global Area 3423965184 bytes
Fixed Size                  2180544 bytes
Variable Size            2566916672 bytes
Database Buffers          838860800 bytes
Redo Buffers               16007168 bytes
```

`cmd`

> rman target /

`rman`

> restore controlfile from 'D:\backup\CON_0KT8TD3F_1_1.CTL';

```
RMAN> restore controlfile from 'D:\backup\CON_0KT8TD3F_1_1.CTL';

Starting restore at 25-JUL-18
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=156 device type=DISK

channel ORA_DISK_1: restoring control file
channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
output file name=D:\APP\ADMINISTRATOR\ORADATA\ORCL\CONTROL01.CTL
output file name=D:\APP\ADMINISTRATOR\FLASH_RECOVERY_AREA\ORCL\CONTROL02.CTL
Finished restore at 25-JUL-18
```

在数据文件目录和闪回目录下会重新生成控制文件

`rman`

> sql 'alter database mount';

```
RMAN> sql 'alter database mount';

sql statement: alter database mount
released channel: ORA_DISK_1
```


`rman`

> restore database;

```
RMAN> restore database;

Starting restore at 25-JUL-18
Starting implicit crosscheck backup at 25-JUL-18
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=156 device type=DISK
Crosschecked 12 objects
Finished implicit crosscheck backup at 25-JUL-18

Starting implicit crosscheck copy at 25-JUL-18
using channel ORA_DISK_1
Finished implicit crosscheck copy at 25-JUL-18

searching for all files in the recovery area
cataloging files...
no files cataloged

using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00001 to D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSTEM01.DBF
channel ORA_DISK_1: restoring datafile 00002 to D:\APP\ADMINISTRATOR\ORADATA\ORCL\SYSAUX01.DBF
channel ORA_DISK_1: restoring datafile 00003 to D:\APP\ADMINISTRATOR\ORADATA\ORCL\UNDOTBS01.DBF
channel ORA_DISK_1: restoring datafile 00004 to D:\APP\ADMINISTRATOR\ORADATA\ORCL\USERS01.DBF
channel ORA_DISK_1: restoring datafile 00005 to D:\APP\ADMINISTRATOR\ORADATA\ORCL\LHCJ_TJ.DBF
channel ORA_DISK_1: reading from backup piece D:\BACKUP\FULL_0IT8TCPN_1_1.BAK
channel ORA_DISK_1: piece handle=D:\BACKUP\FULL_0IT8TCPN_1_1.BAK tag=TAG20180725T172159
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:01:16
Finished restore at 25-JUL-18
```


`rman`

> recover database;

```
RMAN> recover database;

Starting recover at 25-JUL-18
using channel ORA_DISK_1

starting media recovery

unable to find archived log
archived log thread=1 sequence=15
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 07/25/2018 17:55:44
RMAN-06054: media recovery requesting unknown archived log for thread 1 with sequence 15 and starting SCN of 72611696
```

`sqlplus`

> alter database open resetlogs;

```
SQL> alter database open resetlogs;

Database altered.

```

查询下业务表和数据是否正常

> select * from mope.tj_medical;

若正常则数据库恢复完毕。
