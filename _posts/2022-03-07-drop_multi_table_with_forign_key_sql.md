---
title: "پاک کردن تمامی جداول دارای کلید خارجی در SQL Server"
categories:
  - SQL
tags:
  - sql
  - forign_key
  - drop
  - table
---

در یکی از بخش‌های جدا سازی و حرکت به سوی ماکروسرویس، نیاز بود تا از صحت عملکرد سیستم اطمینان حاصل کنیم.  
برای این کار سورس‌های سیستم را در یک سرور دیگر اجرا کردیم و برای بخش دیتابیس نیاز داشتیم تا تیبل‌ها و دیگر عناصر اضافی را حذف کنیم و سپس با دیتاهای واقعی سیستم جدا شده را تست کنیم.  
بدین منظور نیاز بود تا بر روی بکاپ دیتابیس اصلی جدول‌های اضافی که دیتای زیادی نیز داشتند را حذف کنیم.  
انجام کار بالا بصورت دستی زمان‌بر بود پس با کمی جستجو و فکر به کوئری زیر رسیدیم که عناصر مورد نیاز ما در آن باقی می‌مانند و دیگر عناصر که شامل جدول‌ها، Function و storedProcedure هست را حذف می‌کند.  
برای کار بالا نیز نیاز به غیر فعال سازی کلیدهای خارجی یا CONSTRAINT است که در کد زیر درنظر گرفته شده است.  


```sql
USE dbMy;

SET NOCOUNT ON;

DECLARE @views TABLE
               (
                   tablename VARCHAR(255)
               );
INSERT INTO @views
SELECT '[' + SCHEMA_NAME(schema_id) + '].[' + name + ']' AS name
FROM sys.views
WHERE SCHEMA_NAME(schema_id) NOT LIKE '%my%';

DECLARE @dropTableSql NVARCHAR(max);
DECLARE @tablename VARCHAR(255);
DECLARE cTable CURSOR FOR
    SELECT tablename
    FROM @views;

OPEN cTable
FETCH NEXT FROM cTable INTO @tableName
WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @dropTableSql = 'DROP VIEW ' + @tablename + '';
        EXEC sp_executesql @dropTableSql;
        FETCH NEXT FROM cTable INTO @tableName
    END
CLOSE cTable
DEALLOCATE cTable

DECLARE @procedures TABLE
                    (
                        tablename VARCHAR(255)
                    );

INSERT INTO @procedures
SELECT '[' + SCHEMA_NAME(schema_id) + '].[' + name + ']' AS name
FROM sys.procedures
WHERE SCHEMA_NAME(schema_id) NOT LIKE '%my%';

DECLARE cTable CURSOR FOR
    SELECT tablename
    FROM @procedures
OPEN cTable
FETCH NEXT FROM cTable INTO @tableName
WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @dropTableSql = 'DROP PROCEDURE ' + @tablename + '';
        EXEC sp_executesql @dropTableSql;
        FETCH NEXT FROM cTable INTO @tableName
    END
CLOSE cTable
DEALLOCATE cTable

DECLARE @tables TABLE
                (
                    tablename VARCHAR(255)
                );
INSERT INTO @tables
SELECT '[' + SCHEMA_NAME(schema_id) + '].[' + name + ']' AS name
FROM sys.tables
WHERE '[' + SCHEMA_NAME(schema_id) + '].[' + name + ']' NOT IN
      (
       '[my.new].[StateArchive]',
       '[my.new].[Request]'
          );

-- Iterate tables, drop FK constraints and tables
DECLARE cTable CURSOR FOR
    SELECT tablename
    FROM @tables;

OPEN cTable
FETCH NEXT FROM cTable INTO @tableName
WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Identify any FK constraints
        DECLARE @fkCount INT;
        SELECT @fkCount = COUNT(*)

        FROM sys.foreign_keys
        WHERE referenced_object_id = OBJECT_ID(@tablename)

        -- Drop any FK constraints from the table
        IF (@fkCount > 0)
            BEGIN
                DECLARE @dropFkSql NVARCHAR(max);
                SET @dropFkSql = '';
                DECLARE @fkName NVARCHAR(max);

                DECLARE cDropFk CURSOR FOR
                    SELECT 'ALTER TABLE [' + OBJECT_SCHEMA_NAME(parent_object_id) + '].[' +
                           OBJECT_NAME(parent_object_id) + '] DROP CONSTRAINT [' + name + ']',
                           name
                    FROM sys.foreign_keys
                    WHERE referenced_object_id = OBJECT_ID(@tablename)
                OPEN cDropFk
                FETCH NEXT FROM cDropFk INTO @dropfksql,@fkName
                WHILE @@FETCH_STATUS = 0
                    BEGIN
                        EXEC sp_executesql @dropFkSql;

                        SELECT 'Dropped FK Constraint: ' + @fkName
                        FETCH NEXT FROM cDropFk INTO @dropfksql,@fkName
                    END
                CLOSE cDropFk
                DEALLOCATE cDropFk
            END

        -- Drop the table
        SET @dropTableSql = 'DROP TABLE ' + @tablename + '';
        EXEC sp_executesql @dropTableSql;

        SELECT 'Dropped table: ' + @tablename

        FETCH NEXT FROM cTable INTO @tableName
    END
CLOSE cTable
DEALLOCATE cTable

ALTER TABLE [test].[User]
    DROP CONSTRAINT IF EXISTS [CK_Test_User(NationalCode)]

DECLARE @functions TABLE
                   (
                       tablename VARCHAR(255)
                   );

INSERT INTO @functions
SELECT '[' + SCHEMA_NAME(schema_id) + '].[' + name + ']' AS name
FROM sys.objects
WHERE type IN ('FN', 'IF', 'TF')
  AND SCHEMA_NAME(schema_id) NOT LIKE '%my%'
  AND name NOT IN
      (
       'FnGetAccessibleUser',
       'FnGetAccessibleUserByOthersParam'
          )

DECLARE cTable CURSOR FOR
    SELECT tablename
    FROM @functions
OPEN cTable
FETCH NEXT FROM cTable INTO @tableName
WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @dropTableSql = 'DROP FUNCTION ' + @tablename + '';
        EXEC sp_executesql @dropTableSql;
        FETCH NEXT FROM cTable INTO @tableName
    END
CLOSE cTable
DEALLOCATE cTable
```

در کوئری بالا موارد `NOT IN` و `NOT LIKE` مواردی است که می‌خواهیم آنها را در سیستم حفظ کنیم.  

برای ایجاد کوئری پاک کردن تمام کلیدهای خارجی و ایجاد دوباره آنها می‌توانید از کد زیر استفاده کنید

```sql
DECLARE @sql NVARCHAR(MAX) = '';
SELECT @sql = @sql + 'ALTER TABLE ' + QUOTENAME(OBJECT_SCHEMA_NAME(parent_object_id)) + 
                     '.' + QUOTENAME(OBJECT_NAME(parent_object_id)) + 
                     ' DROP CONSTRAINT ' + QUOTENAME(name) + ';' + CHAR(13)
FROM sys.foreign_keys;

SELECT @sql;
```


```sql
DECLARE @sql NVARCHAR(MAX) = '';
SELECT @sql = @sql + 'ALTER TABLE ' + QUOTENAME(OBJECT_SCHEMA_NAME(fk.parent_object_id)) +
                     '.' + QUOTENAME(OBJECT_NAME(fk.parent_object_id)) +
                     ' ADD CONSTRAINT ' + QUOTENAME(fk.name) +
                     ' FOREIGN KEY (' + COL_NAME(fc.parent_object_id, fc.parent_column_id) + ') ' +
                     ' REFERENCES ' + QUOTENAME(OBJECT_SCHEMA_NAME(fk.referenced_object_id)) +
                     '.' + QUOTENAME(OBJECT_NAME(fk.referenced_object_id)) +
                     '(' + COL_NAME(fc.referenced_object_id, fc.referenced_column_id) + ');' + CHAR(13)
FROM sys.foreign_keys AS fk
JOIN sys.foreign_key_columns AS fc ON fk.object_id = fc.constraint_object_id;

SELECT @sql;
```