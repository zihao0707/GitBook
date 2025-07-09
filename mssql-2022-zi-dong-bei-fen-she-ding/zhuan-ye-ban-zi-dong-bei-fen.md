# 專業版自動備份

> 適合有 SQL Server Agent 的 MSSQL Standard / Enterprise 使用者。\
> 透過 SQL Agent Job 每天定時自動備份，並收縮交易記錄。

***

## 🛠️ 建立完整 Job 腳本

```sql
USE msdb;
GO

-- 步驟 1: 創建一個 SQL Server 作業 (Job)
EXEC sp_add_job @job_name = N'AutoBackup_YourDatabase';

-- 步驟 2: 添加備份資料庫和交易記錄的作業步驟
EXEC sp_add_jobstep
  @job_name = N'AutoBackup_YourDatabase',
  @step_name = N'BackupDBAndLog',
  @subsystem = N'TSQL',
  @command = N'
    -- 備份資料庫 (Full Backup)
    BACKUP DATABASE [YourDatabase] 
    TO DISK = N''D:\Backup\YourDatabase_'' + FORMAT(GETDATE(), ''yyyyMMdd'') + ''.bak'' 
    WITH INIT, COMPRESSION, STATS = 10;

    -- 備份交易記錄 (Transaction Log Backup)
    BACKUP LOG [YourDatabase]
    TO DISK = N''D:\Backup\YourDatabase_LOG_'' + FORMAT(GETDATE(), ''yyyyMMdd'') + ''.trn''
    WITH INIT, COMPRESSION, STATS = 10;

    -- 收縮交易日誌檔案 (Shrink the Transaction Log File)
    USE [YourDatabase];
    DBCC SHRINKFILE (N''YourDatabase_log'', 1);
  ';

-- 步驟 3: 添加清理舊備份檔案的作業步驟
EXEC sp_add_jobstep
  @job_name = N'AutoBackup_YourDatabase',
  @step_name = N'CleanOldBackups',
  @subsystem = N'CmdExec',
  @command = N'
    -- 刪除14天前的 .bak 備份檔案
    forfiles /p D:\Backup /m *.bak /d -14 /c "cmd /c del @file"
    
    -- 刪除14天前的 .trn 交易記錄檔案
    forfiles /p D:\Backup /m *.trn /d -14 /c "cmd /c del @file"
  ';

-- 步驟 4: 設定作業排程 (每晚02:00執行)
EXEC sp_add_schedule 
  @schedule_name = N'DailyBackup',
  @freq_type = 4,  -- 每天 (Frequency Type 4 = Daily)
  @active_start_time = 020000;  -- 開始時間: 02:00 AM

-- 步驟 5: 把排程附加到作業
EXEC sp_attach_schedule 
  @job_name = N'AutoBackup_YourDatabase',
  @schedule_name = N'DailyBackup';

-- 步驟 6: 設定作業伺服器
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
