---
title: "مشکل عدم دریافت دیتا بدون هیچ خطایی در زمان فراخوانی Api در حالت https"
categories:
  - Trick
tags:
  - nuxtjs
  - nodejs
  - api
---

یکی از مشکلات عجیبی که در زمان نوشتن یک اپ Nuxtjs با حالت SSR با آن مواجه شدم، عدم دریافت دیتا از api بود.  
بصورت خلاصه مشکل بوجود آمده بصورت زیر بود:  
در حالت تست که هنوز api آماده نشده بود با استفاده از mockoon فراخوانی api ها فیک شده بود که در یک آدرس دلخواه مانند http://localhost:4321 دیتا را برمی‌گرداند.  
بعد از آماده شدن Backend در .net core و زمان اتصال به api واقعی هیچ دیتایی در سمت Frontend دریافت نمی‌شد.  
همچنین در تب Network مرورگر هم هیچ خطایی نبود و همچنین Api مورد نظر کلا فراخوانی نمی‌شد.  
همچنین Front , Back هر دو گواهی ssl تستی داشتند و با https بالا بودند.  
بعد از بررسی‌های بسیار به این نکته رسیدم که در SSR فراخوانی api ها در سمت سرور و توسط nodejs انجام می‌شود و حتی اگر فرانت و بکند شما گواهی ssl هم داشته باشد به دلیل معتبر نبودن واقعی آن فراخوانی api انجام نمی‌شود.  
برای حل این مشکل در زمان Develop کافی است در فایل `.env` خود خط زیر را قرار دهید:  

```
NODE_TLS_REJECT_UNAUTHORIZED=0
```

و یا در فایل `package.json` تغییر زیر را اعمال کنید:  

```
"dev": "set NODE_TLS_REJECT_UNAUTHORIZED=0 && nuxt --env.NODE_TLS_REJECT_UNAUTHORIZED=0",
```

اطلاعات بیشتر درباره این کانفیگ:  

[nodejs](https://nodejs.org/download/release/v10.16.0/docs/api/cli.html#cli_node_tls_reject_unauthorized_value)