# 專業版 UI 設定教學：自動備份與清除

本教學將說明如何透過 **SQL Server Management Studio (SSMS)** 的圖形介面設定 MSSQL 2022 的自動備份與舊檔案清除流程，**無需撰寫 SQL 或 BAT 腳本**。

***

## 💾 1. 設定資料庫備份（.bak）

1. 開啟 SSMS 並連接至你的 SQL Server 實例。
2. 在 **Object Explorer** 中選取你要備份的資料庫。
3. 右鍵點選資料庫 → `Tasks > Back Up...`。
4. 在「Back Up Database」視窗中：
   * 選擇 `Backup type: Full`
   * `Destination: Disk`
5. 點選 `Add...`，輸入備份檔路徑，例如：\
   `D:\Backup\YourDatabase_yyyyMMdd.bak`
6. 勾選 `Overwrite all existing backup sets` 以覆蓋舊備份。
7. 點選 `OK` 執行備份。

***

## 🧾 2. 設定交易記錄備份（.trn）

1. 同樣點選 `Tasks > Back Up...`。
2. `Backup type` 改為 `Transaction Log`。
3. `Destination` 選擇 `Disk`，指定備份檔案路徑，例如：\
   `D:\Backup\YourDatabase_LOG_yyyyMMdd.trn`
4. 點選 `OK` 完成備份。

***

## ⏱️ 3. 設定自動備份作業

使用 SQL Server Agent 排程每日備份與自動清除過期備份。

### ➕ 建立作業（Job）

1. 展開 `SQL Server Agent > Jobs`。
2. 右鍵選擇 `New Job...`，命名為：\
   `AutoBackup_YourDatabase`

### ⚙️ 建立步驟（Steps）

* 點選 `Steps > New...`，新增兩個步驟：

| Step 名稱           | 說明            |
| ----------------- | ------------- |
| `BackupDBAndLog`  | 備份資料庫與交易記錄    |
| `CleanOldBackups` | 刪除 14 天前的備份檔案 |

#### 🔧 CleanOldBackups：命令內容

```bat
forfiles /p D:\Backup /m *.bak /d -14 /c "cmd /c del @file"
forfiles /p D:\Backup /m *.trn /d -14 /c "cmd /c del @file"
```

***

## 🗓️ 4. 建立排程（Schedule）

1. 點選 `Schedules > New...`
2. 設定排程為：

| 項目        | 設定值       |
| --------- | --------- |
| Frequency | Daily（每日） |
| Time      | 02:00 AM  |

***

## ✅ 5. 完成與確認作業

1. 設定完成後點選 `OK` 儲存作業。
2. 確認作業已顯示於 `SQL Server Agent > Jobs` 中。
3. 可右鍵選擇 `Start Job at Step...` 測試作業是否執行成功。

***

## 🗂️ 6. 自動備份流程圖（簡化版）

```scss
每天定時 (SQL Server Agent) →
      ├── 備份資料庫 (.bak)
      ├── 備份交易記錄 (.trn)
      └── 刪除14天前的舊備份檔案
```

***

## 💡 小技巧與注意事項

* ✅ 確保備份路徑所在磁碟有足夠空間。
* ✅ 資料庫的 Recovery Model 須為 `Full` 或 `Bulk-Logged`。
* ✅ 可在備份完成後檢查檔案是否成功產生。

***

這樣設定後，你的 MSSQL 2022 資料庫備份將能每天自動完成，同時定期清除過期備份，有效保護你的資料安全與節省空間！
