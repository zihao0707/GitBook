# 專業版自動備份

> 適合有 SQL Server Agent 的 MSSQL Standard / Enterprise 使用者。\
> 透過 SQL Agent Job 每天定時自動備份，並收縮交易記錄。

***

## 🛠️ 建立完整 Job 腳本

```sql
USE msdb;
GO

EXEC sp_add_job @job_name = N'AutoBackup_YourDatabase';

EXEC sp_add_jobstep
  @job_name = N'AutoBackup_YourDatabase',
  @step_name = N'BackupDBAndLog',
  @subsystem = N'TSQL',
  @command = N'
    BACKUP DATABASE [YourDatabase] 
    TO DISK = N''D:\Backup\YourDatabase_'' + FORMAT(GETDATE(), ''yyyyMMdd'') + ''.bak'' 
    WITH INIT, COMPRESSION, STATS = 10;
    
    BACKUP LOG [YourDatabase]
    TO DISK = N''D:\Backup\YourDatabase_LOG_'' + FORMAT(GETDATE(), ''yyyyMMdd'') + ''.trn''
    WITH INIT, COMPRESSION, STATS = 10;
    
    USE [YourDatabase];
    DBCC SHRINKFILE (N''YourDatabase_log'', 1);
  ';

EXEC sp_add_jobstep
  @job_name = N'AutoBackup_YourDatabase',
  @step_name = N'CleanOldBackups',
  @subsystem = N'CmdExec',
  @command = N'
    forfiles /p D:\Backup /m *.bak /d -14 /c "cmd /c del @file"
    forfiles /p D:\Backup /m *.trn /d -14 /c "cmd /c del @file"
  ';

EXEC sp_add_schedule 
  @schedule_name = N'DailyBackup',
  @freq_type = 4,
  @active_start_time = 020000;

EXEC sp_attach_schedule 
  @job_name = N'AutoBackup_YourDatabase',
  @schedule_name = N'DailyBackup';

EXEC sp_add_jobserver 
  @job_name = N'AutoBackup_YourDatabase';
```

***

## 🔥 流程簡圖

```
每天定時 → SQL Agent Job → 備份 → 收縮 → 清除14天前備份
```

***

## 🛟 常見問題 Q\&A

* Job失敗可設定通知Email
* 收縮頻率建議控制
