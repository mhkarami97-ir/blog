---
title: "انتشار یک Github Package بصورت Nuget"
categories:
  - Nuget
tags:
  - net
  - github
  - nuget
  - package
---

یکی از قابلیت‌های گیتهاب امکان انتشار مستقیم کتابخانه‌های نوشته شده بطور مثال برای .net است که توسط آن می‌توانید nuget package خود را منتشر کنید.  
بدین منظور نیاز است تا یک personal access token در گیتهاب بسازید. برای این کار به بخش زیر بروید:  

 > Settings -> Developer Settings -> Personal Access Tokens > Tokens

سپس بر روی Generate new token و  سپس Generate new token (classic) کلیک کنید.  
در این بخش تیک بخش اول یا همان repo را بزنید.  
سپس تیک دو بخش write:packages , delete:packages را انتخاب کنید.  
می‌توانید Expirations را بر روی never قرار دهید تا کلید ساخته شده منقضی نشود.  
سپس کلید ساخته شده را کپی کنید.  

اکنون اطلاعات اکانت خود را در دستور زیر وارد کنید و سپس آن را در سیستم خود وارد کنید.  

```
dotnet nuget add source https://nuget.pkg.github.com/mhkarami97/index.json --name github-mhkarami97 --username mhkarami97 --password <Your personal Access Token>
```

برای ساخت پکیج نیز می‌توانید از لینک زیر کمک بگیرید.  
[create nuget package](https://learn.microsoft.com/en-us/nuget/quickstart/create-and-publish-a-package-using-visual-studio?tabs=netcore-cli)  

بعد از اینکه کتابخانه خود را نوشتید آن را یکبار بصورت Release بیلد کنید و سپس به آدرس زیر بروید:  

 > /YourProject/bin/Release

در آدرس بالا باید یک فایل با فرمت nupkg مشاهده کنید. در صورتی که وجود نداشت لینک بالا را دوباره مطالعه کنید.  

اکنون کافی است دستور زیر را در این محل در cmd وارد کنید.  

```
dotnet nuget push .\EasyMultiCacheManager.1.0.0.nupkg --api-key <your github access token> --source github-mhkarami97
```

اکنون در ریپازیتوری خود در گیتهاب پکیج مورد نظر را مشاهده می‌کنید.  

[github package](https://github.com/MHKarami97?tab=packages&repo_name=CacheManager)  

در صورتی که می‌خواهید کتابخانه را در سایت Nuget نیز منتشر کنید می‌توانید از لینک زیر کمک بگیرید.  

[publish on nuget](https://learn.microsoft.com/en-us/nuget/nuget-org/publish-a-package)  

نمونه:  
[EasyMultiCacheManager](https://www.nuget.org/packages/EasyMultiCacheManager)  