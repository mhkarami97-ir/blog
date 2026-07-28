---
title: "حل خطا self signed certificate in certificate chain در شبکه سازمانی"
categories:
  - Network
tags:
  - network
  - nodejs
  - ssl
---

اگر پشت شبکه سازمانی (Corporate Proxy) با Node.js کار می‌کنید، احتمالاً با خطاهایی مثل `UNABLE_TO_VERIFY_LEAF_SIGNATURE`، `self signed certificate in certificate chain` یا هشدار `NODE_TLS_REJECT_UNAUTHORIZED` مواجه شده‌اید.  

### چرا این خطا رخ می‌دهد؟
اکثر شبکه‌های سازمانی از تکنولوژی **SSL/TLS Inspection** (مانند Zscaler، Fortinet، Cisco Umbrella) استفاده می‌کنند که در آن یک پروکسی میانی، گواهی سرور واقعی را با گواهی صادرشده توسط CA داخلی سازمان جایگزین می‌کند تا بتواند ترافیک HTTPS را بازرسی کند. مشکل این‌جاست که Node.js به‌طور پیش‌فرض از بسته گواهی داخلی خودش (مبتنی بر Mozilla) استفاده می‌کند، نه از Windows Certificate Store، پس گواهی داخلی سازمان را نمی‌شناسد و اتصال را رد می‌کند.  

### راه‌حل اشتباه که باید از آن پرهیز کرد
خیلی از توسعه‌دهندگان با تنظیم زیر مشکل را «حل» می‌کنند:

```powershell
$env:NODE_TLS_REJECT_UNAUTHORIZED=0
```

این کار **کل مکانیزم اعتبارسنجی TLS** را برای تمام درخواست‌های Node غیرفعال می‌کند و شما را در برابر حملات واقعی MITM آسیب‌پذیر می‌سازد. این روش هرگز نباید به‌صورت دائمی استفاده شود.  

### راه‌حل صحیح: فلگ --use-system-ca
از نسخه Node.js **22.15.0** به بعد، یک فلگ رسمی به نام `--use-system-ca` اضافه شده که به Node اجازه می‌دهد مستقیماً از Windows Certificate Store استفاده کند، بدون این‌که نیاز به export دستی گواهی یا غیرفعال کردن validation باشد.  

#### مرحله ۱: فعال‌سازی دائمی فلگ

```powershell
setx NODE_OPTIONS "--use-system-ca"
```

بعد از این دستور، **حتماً پنجره ترمینال را کاملاً ببندید و یک پنجره جدید باز کنید**؛ چون `setx` مقدار را در Registry ذخیره می‌کند اما فقط پروسه‌های بعدی آن را inherit می‌کنند، نه سشن فعلی.

#### مرحله ۲: تایید فعال بودن فلگ
برای اطمینان از این‌که فلگ واقعاً کار می‌کند، یک فایل تست بسازید:

```javascript
// check-ca.js
const tls = require('tls');

const systemCerts = tls.getCACertificates('system');
const defaultCerts = tls.getCACertificates('default');

console.log('تعداد گواهی سیستم ویندوز:', systemCerts.length);
console.log('تعداد گواهی پیش‌فرض Node (bundled + system):', defaultCerts.length);
```

اجرا با:

```powershell
node check-ca.js
```

اگر عدد `defaultCerts.length` بزرگ‌تر از `systemCerts.length` باشد، یعنی گواهی‌های Windows با موفقیت به لیست پیش‌فرض Node ادغام شده‌اند. طبق مستندات رسمی Node.js، متد `getCACertificates('default')` مجموع (union) گواهی‌های bundled داخلی و گواهی‌های سیستمی است.    

# حل خطا در پایتون

در صورت استفاده از پایتون برای حل خطا ssl در آن کافی است دو دستور زیر را وارد کنید:  

```
pip install --upgrade certifi
```

```
pip install python-certifi-win32
```