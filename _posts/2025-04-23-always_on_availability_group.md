---
title: "مشکل عدم نمایش داده در زمان SELECT از دیتابیس Secondary در SQL Server"
categories:
  - SQL
tags:
  - sql
  - always_on
  - primary
---

در زمانی که از قابلیت `Always On availability group` در SQL Server استفاده می‌کنید و دیتابیس Primary, Secondary دارید در زمان کار با دیتابیس Secondary باید به این نکته دقت کنید که ممکن است حتی اگر ارتباط دو دیتابیس از نوع `Sync` باشد دیتا نهایی در آن وجود نداشته باشد.  
بطور مثال فرض کنید که تعداد زیادی رکورد در جدولی به اسم `Processed` ذخیره می‌کنید و سپس در دیتابیس `ReadOnly` یک کوئری Select برای دریافت پیام‌های پردازش نشده می‌زنید. در این حالت ممکن است رکوردهایی که شما ثبت کرده‌اید هنوز در دیتابیس Secondary نباشد. در این حالت دیتا تکراری را دریافت می‌کنید و در زمان ثبت دوباره آن رکوردها به خطا `Cannot insert duplicate key` بخورید.  
دلیل این مشکل هم در زیر آمده است:  

زمانی که تغییری در Replica اصلی ایجاد می‌شود، این تغییرات به Replica ثانویه ارسال و در فایل لاگ تراکنش نوشته می‌شوند. اما این تغییرات تا زمانی که فرآیند Redo آن‌ها را اعمال نکند، در دیتابیس ثانویه قابل مشاهده نخواهند بود.  
Redo فرآیندی در Replica ثانویه است که لاگ‌های تراکنش دریافتی از Replica اصلی را به صورت فیزیکی روی داده‌های پایگاه داده اعمال می‌کند.  
مراحل کامل:
ثبت تراکنش در Primary: وقتی یک تراکنش (مثلاً INSERT, UPDATE, DELETE) در Replica اصلی اجرا می‌شود، تغییرات در فایل لاگ تراکنش (Transaction Log) ثبت می‌شوند.  
ارسال لاگ به Secondary: این لاگ‌ها در حالت Synchronous یا Asynchronous به Replica ثانویه ارسال می‌شوند.  
ثبت لاگ در Secondary: لاگ‌های دریافتی در فایل لاگ Replica ثانویه ثبت می‌شوند. اما این مرحله فقط ثبت منطقی (logical record) است.  
اجرای Redo در Secondary: یک thread داخلی در SQL Server که به آن Redo Thread می‌گویند، این لاگ‌ها را پروسیس و به‌صورت فیزیکی روی پایگاه داده ثانویه اعمال می‌کند. تا زمانی که Redo این عملیات را انجام ندهد، تغییرات در کوئری‌های SELECT قابل مشاهده نخواهند بود  
حتی در حالت Synchronous Commit، SQL Server منتظر اجرای Redo نمی‌ماند تا تراکنش را Commit کند. بلکه تنها زمانی Commit می‌شود که لاگ به‌درستی در فایل لاگ Replica ثانویه ثبت شده باشد—not Redo.  
به‌همین دلیل ممکن است وضعیت زیر رخ دهد:  
داده در Replica اصلی نوشته شده است.  
لاگ به Replica ثانویه ارسال و ثبت شده است.  
اما چون Redo هنوز اجرا نشده، کوئری SELECT از Replica ثانویه داده‌ی جدید را نمی‌بیند  

راه‌های جلوگیری:  
برای عملیات‌های حساس به داده‌های به‌روز (مثل بررسی وجود داده)، فقط از Replica اصلی استفاده کنید.  
 از Redo Lag Monitoring استفاده کنید. SQL Server امکان مانیتور کردن زمان تأخیر Redo را فراهم کرده است (DMV: sys.dm_hadr_database_replica_states → ستون redo_queue_size و redo_rate)  
 زمان‌بندی اجرای کوئری‌های خواندنی از Replica ثانویه را طوری تنظیم کنید که به فرآیند Redo فرصت اعمال بدهد.  
 از کوئری‌هایی مانند NOT IN یا NOT EXISTS روی Replica ثانویه اجتناب کنید، به‌خصوص وقتی با داده‌های اخیراً درج‌شده سر و کار دارید.  

 اطلاعات بیشتر:  
 [troubleshooting-recovery-queuing-in-alwayson-availability-group](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/availability-groups/troubleshooting-recovery-queuing-in-alwayson-availability-group)