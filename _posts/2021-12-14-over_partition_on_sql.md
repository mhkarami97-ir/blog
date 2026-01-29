---
title: "دستور OVER PARTITION در SQL"
categories:
  - SQL
tags:
  - over
  - partition
  - sql_server
---


```sql
SELECT TOP 100 AVG(Quantity) AS Quantity,
               MAX(Price)    AS Price
FROM [dbo].Decision;
```

که خروجی آن بصورت زیر است:  

| Quantity | Price |
| :--- | :--- |
| 30289 | 5000000000 |


حال اگر بخواهیم شناسه کاربران را هم به خروجی اضافه کنیم با خطا زیر مواجه می‌شویم:  

`[S0001][8120] Line 1: Column 'oms.tse.new.Decision.CustomerId' is invalid in the select list because it is not contained in either an aggregate function or the GROUP BY clause.`

```sql
SELECT TOP 100 CustomerId,
               AVG(Quantity) AS Quantity,
               MAX(Price)    AS Price
FROM [oms.tse.new].Decision;
```

در این مواقع می‌توانیم از دستور `OVER` استفاده کنیم:  

```sql
SELECT TOP 100 CustomerId,
               AVG(Quantity) OVER (PARTITION BY CustomerId)     AS Quantity,
               MAX(Price) OVER (PARTITION BY CustomerId)        AS Price,
               COUNT(CustomerId) OVER (PARTITION BY CustomerId) AS Count
FROM [dbo].Decision;
```

```sql
SELECT TOP 100 CustomerId,
               AVG(Quantity) OVER (PARTITION BY CustomerId)                AS AvgQuantity,
               MAX(Price) OVER (PARTITION BY CustomerId)                   AS MaxPrice,
               COUNT(CustomerId) OVER (PARTITION BY CustomerId)            AS Count,
               ROW_NUMBER() OVER (PARTITION BY Channel ORDER BY Id ASC) AS RowNumber
FROM [dbo].Decision;
```

البته استفاده خیلی نتیجه مفیدی ندارد و روش استفاده بصورت زیر نتیجه مفیدی می‌دهد.  