---
title: "جستجو موارد مشابه در یک ستون String در SQL Server"
categories:
  - Sql
tags:
  - sql
  - find
  - search
---

در صورتی که در دیتابیس SQL Server خود نیاز داشتید بر روی یک ستون VARCHAR جستجو انجام بدهید و مواردی که یک سری موارد خاص را داشتند پیدا کنید می‌توانید از این کوئری استفاده کنید.  
بطور مثال فرض کنید لیستی از شناسه‌ها دارید و می‌خواهید تمام مواردی را که در یکی از ستون‌های آن یکی از این شناسه‌ها وجود داشت پیدا کنید. با این روش می‌توانید این کار را به راحتی انجام دهید.  

```sql
DECLARE @Items NVARCHAR(MAX) = '818068312,818069219';

SELECT *
FROM [dbo].MyTable B
WHERE EXISTS (SELECT 1
              FROM STRING_SPLIT(@Items, ',') AS SplitItems
              WHERE B.Description LIKE '%' + SplitItems.value + '%')
```

برای ایجاد راحت‌تر لیست شناسه‌ها هم می‌توانید از لینک زیر استفاده کنید:  

[select_column_as_single_string_sql](https://blog.mhkarami97.ir/sql/select_column_as_single_string_sql/)