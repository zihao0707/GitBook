# 通用版UI操作自動備份

> 本教學適合「完全不想寫SQL、BAT」的使用者，通過 SQL Server Management Studio (SSMS) 圖形介面完成所有操作！\
> 同時，將展示如何設置「每天備份 .bak 與 .trn」以及「自動清除14天前的備份檔案」。

***

## 🛠️ 步驟一：使用 Management Studio 備份 .bak

1. 打開 **SQL Server Management Studio**
2. 連接到你的主機
3. 右鍵點選你想備份的資料庫，選擇「**Tasks > Back Up...**」
4. 在「備份類型」選擇 **Full**（完整備份）
5. 設定路徑：
   * 加入 **D:\Backup** 目錄
   * 案例檔名設為：`YourDatabase_yyyyMMdd.bak`
6. 勾選「Overwrite all existing backup sets」
7. 點擊 **OK** 完成備份

***

## 🛠️ 步驟二：使用 Management Studio 備份 .trn（交易記錄）

1. 同樣右鍵點選資料庫，選擇「**Tasks > Back Up...**」
2. 在「備份類型」選擇 **Transaction Log**（交易記錄）
3. 設定路徑：
   * 加入 **D:\Backup** 目錄
   * 檔名設為：`YourDatabase_LOG_yyyyMMdd.trn`
4. 點擊 **OK** 完成交易記錄備份

***

## 🛠️ 步驟三：使用 Windows Task Scheduler 自動清除14天前的備份

使用以下簡單的 `.bat` 檔案設定自動清除：

```bat
@echo off
set backupdir=D:\Backup
set retention_days=14

forfiles /p "%backupdir%" /m *.bak /d -%retention_days% /c "cmd /c del @file"
forfiles /p "%backupdir%" /m *.trn /d -%retention_days% /c "cmd /c del @file"
```

**注意：** 必須先確認目錄存在，否則會報錯！

***

## 🚀 完整流程簡圖

```
每天定時 (Task Scheduler) → 執行 backup.bat
      ├── 備份 .bak
      ├── 備份 .trn
      └── 刪除14天前備份
```

***

## 💡 小技巧提醒

* 備份路徑應該選擇有足夠空間的硬碟。
* 確保「恢復模型」設定為 "Full" 或 "Bulk-Logged"，以避免交易記錄過大。
* 每次交易記錄備份後，會自動清除舊的交易記錄檔案（.trn），但如果需要，請啟用「Shrink」動作來手動壓縮。
