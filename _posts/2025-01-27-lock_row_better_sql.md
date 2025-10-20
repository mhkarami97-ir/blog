---
title: "قفل کردن بهینه یک سطر در SQL Server"
categories:
  - SQL
tags:
  - sql
  - lock
  - row
---

در زمان‌هایی که در سمت دیتابیس نیاز دارید که یک سطر خاص را قفل کنید که تا زمان اتمام تراکنش فرد دیگری به آن سطر دسترسی نداشته باشد، یکی از راحت‌ترین کارها باز کردن یک تراکنش Read Committed سمت کد و سپس ایجاد یک SP بصورت زیر است که ابتدا یکی از فیلدهای سطر مورد نظر را آپدیت می‌کند و سپس دیتا مورد نظر شما را برمی‌گرداند.  

```sql
UPDATE [dbo].[Order]
SET LastAccessDateTime = GETDATE()
WHERE Id = @Id

SELECT TOP (1) o.Id,
FROM [dbo].[Order] AS o
          LEFT JOIN [dbo].Customer c WITH (NOLOCK) ON c.Id = o.CustomerCode
WHERE o.Id = @Id
ORDER BY o.Id DESC
```

این کوئری جدول شما را یکبار برای آپدیت و یکبار برای پیدا کردن دیتا می‌خواند. روش بهینه‌تر که باعث بهبود سرعت پردازش شما می‌شود استفاده از کوئری زیر است که با `WITH (XLOCK)` باعث می‌شود همزمان هم سطر مورد نظر قفل شود و هم دیتا برگردد.  
با این روش در حد چند میلی‌ثانیه در تراکنش‌های خود بهبود مشاهده می‌کنید.  

```sql
SELECT TOP (1) o.Id,
FROM [dbo].[Order] AS o WITH (XLOCK)
          LEFT JOIN [dbo].Customer c WITH (NOLOCK) ON c.Id = o.CustomerCode
WHERE o.Id = @Id
ORDER BY o.Id DESC
```