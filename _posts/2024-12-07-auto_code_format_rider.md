---
title: "مرتب‌سازی خودکار کد در Jebtrains Rider هنگام سیو فایل"
categories:
  - Trick
tags:
  - jetbrains
  - rider
  - trick
---

یکی از امکانات خوب Jetbrains IDE ویژگی به اسم Code CleanUp است. با این قابلیت می‌توانید کارهایی مانند مرتب شدن استایل خودکار کدها هنگام ذخیره فایل را انجام دهید.  
برای فعال سازی این قابلیت ابتدا به بخش زیر بروید:  

```
Setting > Editor > Code Cleanup
```
در این بخش می‌توانید یا از آیتم‌های پیش‌فرض استفاده کنید یا با توجه به نیاز خود یک آیتم جدید با تنظیمات دلخواه ایجاد کنید.  

![mhkarami97](/assets/img/rider_cleanup.jpg)  

اکنون به بخش زیر بروید:  

```
Setting > Tools > Action on Save
```
در این بخش کافی است تیک `Reformat and Cleanup Code` را بزنید و از profile آیتمی که در بخش قبل ایجاد کرده بودید را انتخاب کنید.  

![mhkarami97](/assets/img/rider_cleanup1.jpg)  


با تنظیمات بالا هر موقع که فایل شما Save شود پروفایل مربوطه بر روی آن اعمال می‌شود.  

توضیحات بیشتر:  

[Code_Style_Assistance](https://www.jetbrains.com/help/rider/Code_Style_Assistance.html)