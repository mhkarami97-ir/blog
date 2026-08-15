---
title: "خطا Unable to find package بدون هیچی تغییر در زمان بیلد پروژه"
categories:
  - Net
tags:
  - net
  - nuget
  - build
---

اگه سیستم خودتون رو یک روز خاموش کردید و خونه رفتید و روز بعد که اومدید پروژه Net Core شما بیلد نمی‌شد و خطا مشابه زیر رو میداد این پست برای شما هست:  

```
error NU1102:
Unable to find package Microsoft.NETCore.App.Ref with version (= 8.0.30)
- Found 192 version(s) in Public [ Nearest version: 9.0.0-preview.1.24080.9 ]

error NU1102:
Unable to find package Microsoft.AspNetCore.App.Ref with version (= 8.0.30)
- Found 193 version(s) in Public [ Nearest version: 9.0.0-preview.1.24081.5 ]
```

بجای پیکج فوق ممکنه پکیج دیگه‌ای باشه. تشابه خطاها هم این هست که بدون هیچ کاری ظاهر شدن و حتی با پاک کردن کش نوگت سیستم و پوشه‌های bin/obj پروژه هم برطرف نمی‌شن.  

دلیل این خطا آپدیت خودکار ویندوز هست، چیزی که ممکنه بیشتر وقتا بهش توجه نکنیم که چه چیزی رو نصب میکنه. فرض کنید روی سیستم خودتون ورژن‌های زیر از .net core رو داشته باشید:  

```
> dotnet --list-sdks
6.0.428 [C:\Program Files\dotnet\sdk]
8.0.206 [C:\Program Files\dotnet\sdk]
8.0.400 [C:\Program Files\dotnet\sdk]
9.0.205 [C:\Program Files\dotnet\sdk]
10.0.109 [C:\Program Files\dotnet\sdk]
```
و پیامی در ویندوز می‌گیرید که یه path امنیتی آماده نصب هست و شما هم اون رو تایید می‌کنید تا ورژن `10.0.111` از net core نصب بشه.  
اینجا هست که مشکل پیش میاد. اگه فایل `global.json` برای پروژه نداشته باشید بصورت پیش‌فرض آخرین نسخه نصب روی سیستم استفاده میشه و برای پروژه‌ای که از ورژن .net core 8 هم استفاده کرده باشه با همین نسخه آخر بیلد میشه.  
اینجا هست که ممکنه مشکل پیش بیاد و به علت باگی در ورژن منتشر شده نسخه اشتباهی از یک کتابخونه که اینجا `` بود رو برای پروژه بخواد که چون وجود نداره با خطا بیلد مواجه می‌شید.  

راه حل اینه که یا اون ورژن رو پاک کنید تا مشکل برطرف بشه یا فایل global.json رو به پروژه اضافه کنید تا همیشه با یک نسخه مشخص پروژه بیلد بشه:  

```csharp
{
  "sdk": {
    "version": "8.0.400",
    "rollForward": "latestPatch"
  }
}
```


