#如何将Oracle数据从ASM环境恢复到Non-ASM环境？
本文讲述如何将Oracle数据从`RAC+ASM`环境备份的数据恢复到`单机+Non-ASM`的环境。

##第1步 备份
使用`rman`命令将生产机的数据文件和归档日志一起备份。

##第2步 COPY备份文件到到冷备机
假设备份文件在主机`10.20.2.101`的目录`root/nfcq_1020`：

    oracle@yjsdb1:/home/oracle$ cd /oracle/ORABACKUP/database/backupfile

使用scp命令拷贝备份文件到`10.20.1.90`：

    oracle@yjsdb1:/oracle/ORABACKUP/database/backupfile$ scp * 10.20.1.90:/oracle/ORABACKUP/database/backupfile

##第3步 确保pfile文件`inityjsdb.ora`中已增加路径转换的设置

这是其中关键的两行，用来将AMS环境中的路径批量替换为Non-ASM的路径：

    *.db_file_name_convert = ("+DATA","/oradata/yjsdbstb")
    *.log_file_name_convert = ("+DATA","/oradata/yjsdbstb")

完整的pfile文件大概像这样：

    oracle@yjsdb_standby:/oracle/product/11.2.0/dbhome_1/dbs$ cat inityjsdb.ora

    yjsdb.__db_cache_size=1325400064
    yjsdb.__java_pool_size=16777216
    yjsdb.__large_pool_size=16777216
    yjsdb.__oracle_base='/oracle'#ORACLE_BASE set from environment
    yjsdb.__pga_aggregate_target=1459617792
    yjsdb.__sga_target=1828716544
    yjsdb.__shared_io_pool_size=0
    yjsdb.__shared_pool_size=436207616
    yjsdb.__streams_pool_size=0
    *.audit_file_dest='/oracle/admin/yjsdbstb/adump'
    *.audit_trail='db'
    *.compatible='11.2.0.1.0'
    *.control_files='/oradata/yjsdbstb/controlfile/control01.ctl','/oradata/yjsdbstb/controlfile/control02.ctl'#Restore Controlfile
    *.db_block_size=8192
    *.db_domain=''
    *.db_name='yjsdb'
    *.db_recovery_file_dest_size=107374182400
    *.db_recovery_file_dest='/oracle/ORABACKUP/db_recovery_file_dest'
    *.db_file_name_convert = ("+DATA","/oradata/yjsdbstb")
    *.log_file_name_convert = ("+DATA","/oradata/yjsdbstb")
    *.diagnostic_dest='/oracle'
    *.dispatchers='(PROTOCOL=TCP) (SERVICE=yjsdbstbXDB)'
    *.memory_target=3274702848
    *.open_cursors=300
    *.processes=500
    *.remote_login_passwordfile='EXCLUSIVE'
    *.sessions=555
    *.undo_tablespace='UNDOTBS1'

##第4步 恢复数据文件

    # sqlplus '/as sysdba'
    SQL> startup nomount pfile='/oracle/product/11.2.0/dbhome_1/dbs/inityjsdb.ora';
    SQL> exit;

    #rman target /
    RMAN> restore controlfile from '/oracle/ORABACKUP/database/backupfile/dbcontrouljgi201401211';

    RMAN> alter database mount;

    RMAN> catalog start with '/oracle/ORABACKUP/database/backupfile/';

    RMAN> run {
       set newname for database to '/oradata/yjsdbstb/%b';
       restore database;
       switch datafile all;
    }

##第5步 将数据库恢复到最新的归档节点

首先从备份文件中查找最新的归档节点

    RMAN> list backup; 

这可能等待需要一端时间。然后从看到的结果中找到下面的这一段：

    List of Archived Logs in backup set 1296
    Thrd Seq     Low SCN    Low Time            Next SCN   Next Time
    ---- ------- ---------- ------------------- ---------- ---------
    1    27546   712403991  2014-01-21 21:08:09 712593602  2014-01-21 21:46:16
    1    27547   712593602  2014-01-21 21:46:16 712616925  2014-01-21 22:01:05
    1    27548   712616925  2014-01-21 22:01:05 712633321  2014-01-21 22:02:24
    1    27549   712633321  2014-01-21 22:02:24 712640211  2014-01-21 22:03:04
    1    27550   712640211  2014-01-21 22:03:04 712645011  2014-01-21 22:04:02
    1    27551   712645011  2014-01-21 22:04:02 712649740  2014-01-21 22:05:09
    1    27552   712649740  2014-01-21 22:05:09 712655951  2014-01-21 22:07:41
    1    27553   712655951  2014-01-21 22:07:41 712661647  2014-01-21 22:07:43
    1    27554   712661647  2014-01-21 22:07:43 712669829  2014-01-21 22:07:46
    1    27555   712669829  2014-01-21 22:07:46 712679123  2014-01-21 22:07:49
    1    27556   712679123  2014-01-21 22:07:49 712690018  2014-01-21 22:07:53
    1    27557   712690018  2014-01-21 22:07:53 712700383  2014-01-21 22:07:57
    1    27558   712700383  2014-01-21 22:07:57 712705331  2014-01-21 22:08:03
    1    27559   712705331  2014-01-21 22:08:03 712708505  2014-01-21 22:08:04
    1    27560   712708505  2014-01-21 22:08:04 712711914  2014-01-21 22:08:06
    1    27561   712711914  2014-01-21 22:08:06 712714732  2014-01-21 22:08:07
    1    27562   712714732  2014-01-21 22:08:07 712717970  2014-01-21 22:08:09
    1    27563   712717970  2014-01-21 22:08:09 712725314  2014-01-21 22:08:13
    1    27564   712725314  2014-01-21 22:08:13 712736732  2014-01-21 22:29:31
    1    27565   712736732  2014-01-21 22:29:31 712766854  2014-01-21 23:02:51
    1    27566   712766854  2014-01-21 23:02:51 712768335  2014-01-21 23:03:02
    2    26425   712396275  2014-01-21 21:08:06 712617157  2014-01-21 22:01:06
    2    26426   712617157  2014-01-21 22:01:06 712645520  2014-01-21 22:04:08
    2    26427   712645520  2014-01-21 22:04:08 712674631  2014-01-21 22:07:47
    2    26428   712674631  2014-01-21 22:07:47 712702982  2014-01-21 22:08:03
    2    26429   712702982  2014-01-21 22:08:03 712722478  2014-01-21 22:08:12
    2    26430   712722478  2014-01-21 22:08:12 712766865  2014-01-21 23:02:53
    2    26431   712766865  2014-01-21 23:02:53 712768339  2014-01-21 23:03:04

线程1中的最大序列是27566，线程2中的最大序列是26431，因此我们取27566，再加1即27567，作为最新的归档节点。

现在执行恢复下面的脚本：

    RMAN> run {
      set until sequence 27567 thread 1;
      recover database;
    }

至此，若全程无任何报错，数据就应已正常恢复。

##第6步 修改已恢复数据中的日志文件路径
（若无此步骤，启动时将出现ORA-03113错误）

列出已恢复数据中的日志文件路径：

    # sqlplus '/as sysdba'
    SQL> select member from v$logfile;

    '+DATA/yjsdb/onlinelog/group_1.258.822930965'
    '+DATA/yjsdb/onlinelog/group_1.259.822930965'
    '+DATA/yjsdb/onlinelog/group_2.260.822930967'
    '+DATA/yjsdb/onlinelog/group_2.261.822930967'
    '+DATA/yjsdb/onlinelog/group_3.268.822934461'
    '+DATA/yjsdb/onlinelog/group_3.269.822934463'
    '+DATA/yjsdb/onlinelog/group_4.270.822934463'
    '+DATA/yjsdb/onlinelog/group_4.271.822934463'

根据列出的结果，编辑一组文件。编辑时，请注意观察文件名对应的规则（提示，根据group分组^_^）

下面是规则示例：

    group_1.258.822930965 － redo0101.log
    group_1.259.822930965 － redo0102.log
    group_2.260.822930967 － redo0201.log
    group_2.261.822930967 － redo0202.log

下面是完整的文件：

    alter database rename file "+DATA/yjsdb/onlinelog/group_1.258.822930965" to "/oradata/yjsdbstb/redo0101.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_1.259.822930965" to "/oradata/yjsdbstb/redo0202.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_2.260.822930967" to "/oradata/yjsdbstb/redo0201.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_2.261.822930967" to "/oradata/yjsdbstb/redo0202.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_3.268.822934461" to "/oradata/yjsdbstb/redo0301.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_3.269.822934463" to "/oradata/yjsdbstb/redo0302.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_4.270.822934463" to "/oradata/yjsdbstb/redo0401.log";
    alter database rename file "+DATA/yjsdb/onlinelog/group_4.271.822934463" to "/oradata/yjsdbstb/redo0402.log";

先将上述命令逐条贴到sqlplus窗口执行，若无报错，则可将剩下的指令一起贴上去。

若逐条贴时报错，可重新查询v$logfile，若已修改，则不必理会，继续逐条执行。`报错的原因可能是sqlplus对'+DATA'中的'+'符号敏感，但仍会执行这条语句。`


第7步 重新启动恢复的数据库

