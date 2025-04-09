# MSSQL 2019 自動補齊預設值 & 不允許NULL 欄位



### 📄 說明 <a href="#shuo-ming" id="shuo-ming"></a>

本文件記錄 MSSQL 2019 中，自動尋找資料庫所有未設定預設值的欄位，並依照型態給予預設值，同時將允許 NULL 的欄位改為「不允許 NULL」，並且記錄處理過程 Log 的完整腳本與流程說明。

### ⚙ 功能總覽 <a href="#gong-neng-zong-lan" id="gong-neng-zong-lan"></a>

1\. 自動找出未設定 DEFAULT 的欄位。

2\. 依據欄位型態，設定適當的 DEFAULT 值（如 nvarchar、int、decimal）。

3\. 將允許 NULL 的欄位改為「NOT NULL」。

4\. 自動產生並執行 ALTER TABLE 指令。

5\. 全程記錄處理的每一筆欄位動作與時間戳記。

### 🛠 腳本流程 <a href="#jiao-ben-liu-cheng" id="jiao-ben-liu-cheng"></a>

1\. 掃描資料庫內所有 BASE TABLE。

2\. 篩選出無 DEFAULT 且型態符合設定 (nvarchar、int、decimal) 的欄位。

3\. 排除主鍵與自動增號 (IDENTITY) 欄位。

4\. 依資料型態決定 DEFAULT 值並設定。

5\. 如欄位允許 NULL，修改成 NOT NULL。

6\. 每次執行記錄 ALTER 指令與執行時間。

### 📋 注意事項 <a href="#zhu-yi-shi-xiang" id="zhu-yi-shi-xiang"></a>

⚡ 修改成 NOT NULL 前，請確保欄位內部資料無 NULL，避免失敗。

⚡ 預設值設定以('')、((0))為例，依需求可自訂。

⚡ 使用本腳本建議事前完整備份資料庫！

### ⚙ 先建立SchemaChangeLog 資料表，紀錄LOG <a href="#xian-jian-li-schemachangelog-zi-liao-biao-ji-lu-log" id="xian-jian-li-schemachangelog-zi-liao-biao-ji-lu-log"></a>

Copy

```
IF OBJECT_ID('dbo.SchemaChangeLog', 'U') IS NULL
BEGIN
    CREATE TABLE dbo.SchemaChangeLog (
        LogID INT IDENTITY(1,1) PRIMARY KEY,
        ChangeTime DATETIME DEFAULT(GETDATE()),
        TableName NVARCHAR(128),
        ColumnName NVARCHAR(128),
        ActionType NVARCHAR(100),
        SqlCommand NVARCHAR(MAX),
        Operator NVARCHAR(50) DEFAULT (SUSER_SNAME())  -- 執行人
    );
    PRINT '✅ 建立 SchemaChangeLog 成功！';
END
ELSE
BEGIN
    PRINT '📝 SchemaChangeLog 已存在，略過建立。';
END
```

### ⚙執行下列語法 <a href="#zhi-xing-xia-lie-yu-fa" id="zhi-xing-xia-lie-yu-fa"></a>

Copy

```
-- ==== 設定區 ====
DECLARE @TargetTable NVARCHAR(128) = NULL; -- 指定資料表，例如 'MET03_0000'，NULL = 全部
DECLARE @ExecuteCommands BIT = 0;           -- 1=立即執行, 0=只產生語法

-- ==== 宣告變數 ====
DECLARE @TableName NVARCHAR(128);
DECLARE @ColumnName NVARCHAR(128);
DECLARE @DataType NVARCHAR(128);
DECLARE @IsNullable NVARCHAR(3);
DECLARE @CharacterMaximumLength INT;
DECLARE @NumericPrecision INT;
DECLARE @NumericScale INT;
DECLARE @DefaultValue NVARCHAR(100);
DECLARE @SqlCmd NVARCHAR(MAX);
DECLARE @AlterNullCmd NVARCHAR(MAX);
DECLARE @Now DATETIME = GETDATE();
DECLARE @ErrMsg NVARCHAR(MAX);

-- ==== 暫存要處理的欄位 ====
IF OBJECT_ID('tempdb..#ColumnsToModify') IS NOT NULL
    DROP TABLE #ColumnsToModify;

SELECT 
    a.TABLE_NAME AS TableName,
    b.COLUMN_NAME AS ColumnName,
    b.DATA_TYPE AS DataType,
    b.IS_NULLABLE AS IsNullable,
    b.CHARACTER_MAXIMUM_LENGTH AS CharacterMaximumLength,
    b.NUMERIC_PRECISION AS NumericPrecision,
    b.NUMERIC_SCALE AS NumericScale
INTO #ColumnsToModify
FROM INFORMATION_SCHEMA.TABLES a
JOIN INFORMATION_SCHEMA.COLUMNS b ON a.TABLE_NAME = b.TABLE_NAME
WHERE a.TABLE_TYPE = 'BASE TABLE'
  AND b.COLUMN_DEFAULT IS NULL
  AND b.DATA_TYPE IN ('nvarchar', 'int', 'decimal')
  AND a.TABLE_NAME + '-' + b.COLUMN_NAME NOT IN (
        SELECT TABLE_NAME + '-' + COLUMN_NAME 
        FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
    )
  AND a.TABLE_NAME + '-' + b.COLUMN_NAME NOT IN (
        SELECT b.name + '-' + a.name 
        FROM sys.identity_columns a 
        INNER JOIN sys.objects b ON a.object_id = b.object_id
    )
  AND (@TargetTable IS NULL OR a.TABLE_NAME = @TargetTable)
ORDER BY a.TABLE_NAME, b.COLUMN_NAME;

-- ==== 開啟 Transaction ====
IF @ExecuteCommands = 1
BEGIN
    BEGIN TRANSACTION;
END

-- ==== 處理每一筆 ====
WHILE EXISTS (SELECT 1 FROM #ColumnsToModify)
BEGIN
    BEGIN TRY
        -- 抓一筆資料
        SELECT TOP 1
            @TableName = TableName,
            @ColumnName = ColumnName,
            @DataType = DataType,
            @IsNullable = IsNullable,
            @CharacterMaximumLength = CharacterMaximumLength,
            @NumericPrecision = NumericPrecision,
            @NumericScale = NumericScale
        FROM #ColumnsToModify;

        -- 設定預設值
        SET @DefaultValue = 
            CASE 
                WHEN @DataType = 'nvarchar' THEN N'('''')'
                WHEN @DataType = 'int' THEN N'((0))'
                WHEN @DataType = 'decimal' THEN N'((0))'
                ELSE NULL
            END;

        -- 組成 ALTER DEFAULT 語法
        SET @SqlCmd = 
            N'ALTER TABLE [' + @TableName + N'] ADD DEFAULT ' + @DefaultValue +
            N' FOR [' + @ColumnName + N'] WITH VALUES;';

        -- 顯示SQL
        RAISERROR('%s', 0, 1, @SqlCmd) WITH NOWAIT;

        IF @ExecuteCommands = 1
        BEGIN
            EXEC sp_executesql @SqlCmd;
            -- 插入 Log
            INSERT INTO dbo.SchemaChangeLog (ChangeTime, TableName, ColumnName, ActionType, SqlCommand)
            VALUES (GETDATE(), @TableName, @ColumnName, N'Add Default', @SqlCmd);
        END

        -- 如果允許 NULL，組成 ALTER COLUMN 改 NOT NULL
        IF @IsNullable = 'YES'
        BEGIN
            SET @AlterNullCmd = 
                N'ALTER TABLE [' + @TableName + N'] ALTER COLUMN [' + @ColumnName + N'] ' +
                CASE 
                    WHEN @DataType = 'nvarchar' THEN 'NVARCHAR(' + 
                        CASE WHEN @CharacterMaximumLength = -1 THEN 'MAX' ELSE CAST(@CharacterMaximumLength AS NVARCHAR) END + ')'
                    WHEN @DataType = 'int' THEN 'INT'
                    WHEN @DataType = 'decimal' THEN 'DECIMAL(' + 
                        CAST(@NumericPrecision AS NVARCHAR) + ',' + CAST(@NumericScale AS NVARCHAR) + ')'
                    ELSE @DataType
                END +
                ' NOT NULL;';

            RAISERROR('%s', 0, 1, @AlterNullCmd) WITH NOWAIT;

            IF @ExecuteCommands = 1
            BEGIN
                EXEC sp_executesql @AlterNullCmd;
                -- 插入 Log
                INSERT INTO dbo.SchemaChangeLog (ChangeTime, TableName, ColumnName, ActionType, SqlCommand)
                VALUES (GETDATE(), @TableName, @ColumnName, N'Alter Not Null', @AlterNullCmd);
            END
        END

        -- 刪除已處理
        DELETE FROM #ColumnsToModify
        WHERE TableName = @TableName AND ColumnName = @ColumnName;
        
    END TRY
    BEGIN CATCH
        SET @ErrMsg = ERROR_MESSAGE();
        PRINT '⚠️ 錯誤: ' + @ErrMsg;
        IF @ExecuteCommands = 1
        BEGIN
            ROLLBACK TRANSACTION;
            PRINT '❌ 發生錯誤，自動 Rollback！';
            RETURN;
        END
    END CATCH
END

-- ==== 提交 Transaction ====
IF @ExecuteCommands = 1
BEGIN
    COMMIT TRANSACTION;
    PRINT '✅ 全部成功，已 Commit！';
END

-- ==== 顯示完成訊息 ====
DECLARE @EndTime NVARCHAR(50) = FORMAT(GETDATE(), 'yyyy-MM-ddTHH:mm:ss.fffffffzzz');
PRINT '🎉 處理完成！完成時間: ' + @EndTime;
```
