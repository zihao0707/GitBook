# MSSQL 2022 免費版自動備份與清除教學【黑暗酷版】

> 本教學適合「無SQL Server Agent」的 MSSQL Express / 免費版使用者。\
> 透過 Windows 排程器搭配 .bat + .sql 實現「每日自動備份 .bak 與 .trn」。\
> 同時自動清除超過14天的舊備份。

***

## 🛠️ 步驟一：建立備份 SQL 腳本

在 `D:\Scripts\backup_database.sql` 建立以下內容：

```sql
-- 備份完整資料庫
BACKUP DATABASE [YourDatabase]
TO DISK = N'D:\Backup\YourDatabase_$(ESCAPE_SQUOTE(DATEPART(yyyy, GETDATE())))$(ESCAPE_SQUOTE(FORMAT(GETDATE(), 'MMdd'))).bak'
WITH INIT, COMPRESSION, STATS = 10;

-- 備份交易記錄
BACKUP LOG [YourDatabase]
TO DISK = N'D:\Backup\YourDatabase_LOG_$(ESCAPE_SQUOTE(DATEPART(yyyy, GETDATE())))$(ESCAPE_SQUOTE(FORMAT(GETDATE(), 'MMdd'))).trn'
WITH INIT, COMPRESSION, STATS = 10;

-- 收縮交易記錄檔
USE [YourDatabase];
DBCC SHRINKFILE (N'YourDatabase_log', 1);
```

## 🛠️ 步驟二：建立備份批次檔 .bat

在 `D:\Scripts\backup_database.bat` 建立以下內容：

```bat
@echo off
set dbname=YourDatabase
set backupdir=D:\Backup
set retention_days=14

sqlcmd -S localhost -E -i "D:\Scripts\backup_database.sql"

forfiles /p "%backupdir%" /m *.bak /d -%retention_days% /c "cmd /c del @file"
forfiles /p "%backupdir%" /m *.trn /d -%retention_days% /c "cmd /c del @file"
```

## 🛠️ 步驟三：設定Windows排程器

1. 開啟「工作排程器」
2. 新增基本工作：每天執行
3. 動作設定：啟動程式，選擇 `.bat`

***

## 🔥 流程簡圖

```
每天定時 → 執行 .bat → 呼叫 .sql → 備份 .bak/.trn → 收縮 → 刪除舊檔
```

***

## 🛟 常見問題 Q\&A

* 資料夾不存在需手動建立
* 資料庫過大收縮可能無效
