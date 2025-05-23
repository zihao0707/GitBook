# 拆解跨日資料 以遞迴方式為例

此 SQL 查詢用於處理生產工時資料，將跨日的工作紀錄依據每一天拆分，並計算每日實際工時（以分鐘為單位）。

***

## 📌 功能目標

* 將跨日工時拆解為每日記錄。
* 針對每筆資料，自動補上未填 `date_e` 的情況（使用當天日期）。
* 可處理最長 30 天跨日的資料。
* 適合建立為檢視表，供報表或 BI 系統查詢使用。

***

## 📊 拆解邏輯圖示

```plaintext
原始資料：
┌────────────┬────────────┐
│ 起始時間    │ 結束時間     │
├────────────┼────────────┤
│ 03/31 16:53│ 04/01 08:43│
└────────────┴────────────┘

拆解為：
┌──────┬────────┬────────┐
│ 日期  │ 起始時間│ 結束時間 │
├──────┼────────┼────────┤
│ 03/31│ 16:53  │ 23:59  │
│ 04/01│ 00:00  │ 08:43  │
└──────┴───────-┴────────┘
```

***

## 🧠 遞迴邏輯說明

此查詢使用 CTE 遞迴（Common Table Expression）方式處理跨日資料：

1. **第一層 CTE**：提取原始資料中第一天的資料。
2. **遞迴層**：每日 +1，直到跨日結束時間為止，每次新增一筆該日資料。

***

## 💡 SQL 原始碼

```sql
WITH RecursiveSplit AS (
    -- 第一天
    SELECT
        work_code,
        station_code,
        mac_code,
        CAST(date_s AS DATE) AS work_date,
        CAST(date_s AS DATETIME) + CAST(time_s AS DATETIME) AS start_time,
        CASE 
            WHEN DATEDIFF(DAY, date_s, ISNULL(date_e, GETDATE())) = 0 
                THEN CAST(ISNULL(date_e, GETDATE()) AS DATETIME) + CAST(time_e AS DATETIME)
            ELSE CAST(date_s AS DATETIME) + '23:59:59'
        END AS end_time,
        CAST(date_s AS DATETIME) + CAST(time_s AS DATETIME) AS original_start_datetime,
        CAST(ISNULL(date_e, GETDATE()) AS DATETIME) + CAST(time_e AS DATETIME) AS original_end_datetime
    FROM MED08_0000
    WHERE DATEDIFF(DAY, date_s, ISNULL(date_e, GETDATE())) <= 30

    UNION ALL

    -- 遞迴：每天往後加，直到結束日
    SELECT
        r.work_code,
        r.station_code,
        r.mac_code,
        DATEADD(DAY, 1, r.work_date) AS work_date,
        CAST(DATEADD(DAY, 1, r.work_date) AS DATETIME) AS start_time,
        CASE 
            WHEN DATEADD(DAY, 1, r.work_date) = CAST(r.original_end_datetime AS DATE)
                THEN r.original_end_datetime
            ELSE CAST(DATEADD(DAY, 1, r.work_date) AS DATETIME) + '23:59:59'
        END AS end_time,
        r.original_start_datetime,
        r.original_end_datetime
    FROM RecursiveSplit r
    WHERE DATEADD(DAY, 1, r.work_date) <= CAST(r.original_end_datetime AS DATE)
)

-- 最終輸出：包含每天工時
SELECT
    work_code,
    station_code,
    mac_code,
    CONVERT(VARCHAR(10), work_date, 111) AS work_date, 
    CONVERT(VARCHAR(8), start_time, 108) AS time_s,
    CONVERT(VARCHAR(8), end_time, 108) AS time_e,
    DATEDIFF(MINUTE, start_time, end_time) AS work_time
FROM RecursiveSplit
```

***

### 🧠 功能說明

#### ➤ 初始資料來源

資料來自 `MED08_0000` 表格，欄位包含：

* `work_code`: 工單代碼
* `station_code`: 工位代碼
* `mac_code`: 機台代碼
* `date_s` + `time_s`: 工時開始日期與時間
* `date_e` + `time_e`: 工時結束日期與時間（可為空）

#### ➤ 處理邏輯

使用 CTE（Common Table Expression）結構中的 **遞迴查詢** 來拆分每筆跨日資料：

```sql
sqlCopyEditWITH RecursiveSplit AS (
  -- 初始資料 (第1天)
  ...
  UNION ALL
  -- 遞迴：每天往後加
  ...
)
```

1. **初始 SELECT**：產出第一天的工時區段
2. **遞迴 SELECT**：依照日期一天一天往後展開，直到原始結束日
3. **最終 SELECT**：輸出每天拆分後的紀錄與該天的工時長度（以分鐘計）

***

### 📘 欄位說明

| 欄位名稱           | 說明                     |
| -------------- | ---------------------- |
| `work_code`    | 工單代碼                   |
| `station_code` | 工位代碼                   |
| `mac_code`     | 機台代碼                   |
| `work_date`    | 拆分後的工作日（格式：yyyy/mm/dd） |
| `time_s`       | 當日工作起始時間（格式：HH:mm:ss）  |
| `time_e`       | 當日工作結束時間（格式：HH:mm:ss）  |
| `work_time`    | 當日工作時數（以分鐘計）           |

***

### 📌 範例輸出

| work\_code | work\_date | time\_s  | time\_e  | work\_time |
| ---------- | ---------- | -------- | -------- | ---------- |
| A123       | 2025/05/23 | 16:53:01 | 23:59:59 | 427        |
| A123       | 2025/05/24 | 00:00:00 | 08:43:10 | 523        |

## 🛠️ 可優化項目建議

| 項目     | 建議                                          |
| ------ | ------------------------------------------- |
| 最大遞迴深度 | 若可能跨超過 100 天，考慮使用 `OPTION (MAXRECURSION 0)` |
| 效能     | 建議在 `date_s`, `date_e`, `work_code` 建立索引    |
| 檢視表用途  | 將此查詢建立為 VIEW 可供報表直接查詢                       |
| 篩選條件   | 可增加條件排除異常資料，如 `time_s > time_e`             |

***

## 📎 使用建議

```sql
CREATE VIEW V_DailyWorkTimeSplit AS
-- [貼上上述查詢語法]
```

***
