---
title: "فعال سازی RedGate SQL Change Automation"
categories:
  - Trick
tags:
  - trick
  - sql
  - crack
---

اگر از محصولات RedGate برای کار با دیتابیس استفاده کرده باشید احتمال زیاد در زمان فعال سازی آنها به مشکل برخورد کرده‌اید.  
از عدم امکان فعال سازی دستی تا فعال نشدن محصول حتی با وارد کردن سریال.  
برای حل این مشکلات می‌توانید کارهای زیر را انجام دهید:  

ابتدا مطمئن شوید در فایل host ویندوز موارد زیر وجود دارد:  

```
172.20.24.86 www.red-gate.com
127.0.0.1 featureusage.red-gate.com
127.0.0.1 download.red-gate.com
127.0.0.1 licensing.red-gate.com
127.0.0.1 update.red-gate.com
127.0.0.1 productlogin.red-gate.com
127.0.0.1 permits.red-gate.com
127.0.0.1 identityprovider.red-gate.com
```

نکته مهم دیگر این است که یک سری از ابزارها نسخه `Pro` , `Professional` ندارند و فقط با سریال `Standard` می‌توان آنها را فعال کرد.  
اما نکته این است که اگر آنها را با نسخه pro فعال کرده باشید خطایی نمی‌دهد اما ابزار نیز باز نمی‌شود.  
بطور مثال `SQL Change Automation` فقط با standard فعال می‌شود.  
برای این کار کافی است در ابزار فعال ساز حالت standard را انتخاب کنید.  

![red-gate](/assets/img/red-gate.png)  

اگر از قبل فعال سازی کرده‌اید اما به خطا خورده است می‌توانید از آدرس زیر یکبار سریال ابزار مورد نظر با فرمت `lic` را پاک کنید و بعد از بستن و دوباره باز کردن `SQL Server Management Studio` ابزار فوق را فعال سازی کنید.  

> C:\ProgramData\Red Gate\Licenses