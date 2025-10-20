---
title: "خروجی کوئری بصورت Json در SQL Server"
categories:
  - Sql
tags:
  - sql
  - json
  - select
---

توسط `FOR JSON PATH` در Sql Server می‌توانید به راحتی خروجی کوئری خود را بصورت Json دربیاورید و از آن در برنامه خود استفاده کنید.  

```sql
SELECT *
FROM [dbo].MyTable
FOR JSON PATH;
```

```sql
SELECT *
FROM [dbo].MyTable
FOR JSON PATH, INCLUDE_NULL_VALUES;
```

```sql
SELECT *
FROM [dbo].MyTable
FOR JSON PATH, INCLUDE_NULL_VALUES, WITHOUT_ARRAY_WRAPPER;
```

```sql
SELECT *
FROM [dbo].MyTable
FOR JSON PATH, , ROOT('Results');
```

برای ایجاد راحت‌تر لیست شناسه‌ها هم می‌توانید از لینک زیر استفاده کنید:  

[format-query-results-as-json-with-for-json-sql-server](https://learn.microsoft.com/en-us/sql/relational-databases/json/format-query-results-as-json-with-for-json-sql-server)