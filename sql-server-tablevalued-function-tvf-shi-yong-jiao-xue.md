# SQL Server Table-Valued Function (TVF) 使用教學

## 📌 什麼是 Table-Valued Function？

Table-Valued Function（TVF）是 SQL Server 中一種可以「回傳表格資料」的函數，適合用來：

* 包裝複雜查詢邏輯
* 支援參數化查詢
* 提升查詢重用性與可讀性

***

## 📂 章節目錄

* [建立 Table-Valued Function](sql-server-tablevalued-function-tvf-shi-yong-jiao-xue.md#建立-table-valued-function)
* [查詢使用方式](sql-server-tablevalued-function-tvf-shi-yong-jiao-xue.md#查詢使用方式)
* [修改 Function](sql-server-tablevalued-function-tvf-shi-yong-jiao-xue.md#修改-function)
* [刪除 Function](sql-server-tablevalued-function-tvf-shi-yong-jiao-xue.md#刪除-function)
* [範例解說：查詢工單每日數量](sql-server-tablevalued-function-tvf-shi-yong-jiao-xue.md#範例解說查詢工單每日數量)

***

## ✅ 建立 Table-Valued Function

基本語法如下：

```sql
CREATE FUNCTION [函數名稱] (
    @參數1 資料型別,
    @參數2 資料型別
)
RETURNS TABLE
AS
RETURN
    SELECT ... -- 回傳表格資料查詢
```

***

## 🔍 查詢使用方式

```sql
SELECT * FROM [函數名稱](@參數值1, @參數值2)
```

你可以像查表一樣再加上 WHERE、JOIN、ORDER BY 等操作。

***

## ✏️ 修改 Function

使用 `ALTER FUNCTION` 修改內容：

```sql
ALTER FUNCTION [函數名稱] (
    @參數1 資料型別,
    @參數2 資料型別
)
RETURNS TABLE
AS
RETURN
    SELECT ...
```

***

## ❌ 刪除 Function

刪除函數需使用：

```sql
DROP FUNCTION [函數名稱]
```

***

## 📘 範例解說：查詢工單每日數量

### 📌 函數名稱：`fn_MEI09_Daily_Qty_Filtered`

此函數會根據輸入的機台代號（`mac_code`）與工單代號（`wrk_code`），查詢當天與昨天的數量，並計算總和。

```sql
CREATE FUNCTION fn_MEI09_Daily_Qty_Filtered(
    @mac_code VARCHAR(20),
    @wrk_code VARCHAR(20)
)
RETURNS TABLE
AS
RETURN
SELECT DISTINCT
    v.wrk_code,
    v.mac_code,
    ISNULL(t.today_qty, 0) AS today_qty,
    ISNULL(tm.tomorrow_qty, 0) AS tomorrow_qty,
    CASE 
        WHEN ISNULL(t.today_qty, 0) = 0 THEN (
            SELECT ISNULL(SUM(pro_qty), 0) 
            FROM MED09_0000 
            WHERE ins_date = CONVERT(varchar(10), GETDATE(), 111)
              AND mac_code = v.mac_code 
              AND wrk_code = v.wrk_code
        )
        ELSE ISNULL(t.today_qty, 0) + ISNULL(tm.tomorrow_qty, 0)
    END AS total_qty
FROM 
    v_OK_QTY_WRK_DAY v
LEFT JOIN (
    SELECT wrk_code, mac_code, SUM(today_qty) AS today_qty
    FROM MEI09_0000
    WHERE ins_date = CONVERT(varchar(10), GETDATE(), 111)
    GROUP BY wrk_code, mac_code
) t ON v.wrk_code = t.wrk_code AND v.mac_code = t.mac_code
LEFT JOIN (
    SELECT wrk_code, mac_code, SUM(tomorrow_qty) AS tomorrow_qty
    FROM MEI09_0000
    WHERE ins_date = CONVERT(varchar(10), DATEADD(DAY, -1, GETDATE()), 111)
    GROUP BY wrk_code, mac_code
) tm ON v.wrk_code = tm.wrk_code AND v.mac_code = tm.mac_code
WHERE 
    v.ins_date >= CONVERT(varchar(10), DATEADD(DAY, -1, GETDATE()), 111)
    AND v.mac_code = @mac_code
    AND v.wrk_code = @wrk_code

```

***

### 🔎 查詢方式

```sql
SELECT * FROM fn_MEI09_Daily_Qty_Filtered('E1G-099', 'W123')
```

***

### 🛠 修改版本（支援指定日期）

如果希望支援指定日期查詢，可以再加一個參數：

```sql
ALTER FUNCTION fn_MEI09_Daily_Qty_Filtered(
    @mac_code VARCHAR(20),
    @wrk_code VARCHAR(20),
    @target_date DATE
)
RETURNS TABLE
AS
RETURN
SELECT ...
-- 把 GETDATE() 改成 @target_date
```

***

### 🧹 刪除函數

```sql
DROP FUNCTION fn_MEI09_Daily_Qty_Filtered
```

***

## 📎 小提醒

* TVF 不允許 `BEGIN...END` 區塊，只能回傳單一 SELECT 查詢。
* 不支援暫存表、動態 SQL。
* 適合包裝資料查詢邏輯，不建議處理複雜的流程控制。

***

## 📤 匯出與應用

你可以：

* 匯出這份 `.md` 文件至 GitBook 或 Markdown 編輯器
* 用於團隊文件、自動化部署文件等
