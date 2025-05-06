# 使用EXISTS查詢已匯入與未匯入的資料

## 📁 資料背景

我們有一批 `mo_code` 資料，格式為 `001XXXXXXXXX`（共 12 碼），已匯入部分資料至資料表 `MET01_0000`。現在要比對這批資料中哪些已經存在於資料表中，哪些尚未存在。

***

## ✅ 查詢 **已存在的 mo\_code**

```sql
SELECT existing.mo_code
FROM (
    VALUES
        ('001100128466'),
        ('001100128467'),
        ('001100128468'),
        ('001100128473')
) AS existing(mo_code)
WHERE EXISTS (
    SELECT 1
    FROM MET01_0000 AS m
    WHERE m.mo_code = existing.mo_code
);
```

***

## ❌ 查詢 **尚未存在的 mo\_code**

```sql
SELECT missing.mo_code
FROM (
    VALUES
        ('001100128466'),
        ('001100128467'),
        ('001100128468'),
        ('001100128473'),
        ('001100128474')
) AS missing(mo_code)
WHERE NOT EXISTS (
    SELECT 1
    FROM MET01_0000 AS m
    WHERE m.mo_code = missing.mo_code
);
```

***

## 🧠 說明與應用

| 部分             | 說明                                            |
| -------------- | --------------------------------------------- |
| `VALUES (...)` | 建立暫時的資料集合供比對使用                                |
| `EXISTS`       | 判斷某筆 `mo_code` 是否存在於資料表中                      |
| `NOT EXISTS`   | 判斷某筆 `mo_code` 是否尚未存在於資料表中                    |
| 用途範例           | <p>- 驗證匯入資料是否完整<br>- 執行補充匯入作業<br>- 製作驗證報表</p> |

***

如需進一步整合此語法進 ETL 或自動化驗證流程，也可以考慮將 `VALUES` 部分提取為暫存表或中介表，以利資料追蹤與維護。
