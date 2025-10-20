---
title: "استفاده از Static Constructors"
categories:
  - Net
tags:
  - constructors
  - net
  - static
---

توسط قابلیت Static Constructor می‌توانید مقداردهی اولیه به اعضای استاتیک کلاس یا انجام تنظیمات اولیه‌ای که نیاز به یک بار اجرا دارند، استفاده کنید.  
بطور مثال فرض کنید در کتابخانه Flurl می‌خواهید استفاده از NewtonSoft را فقط یکبار انجام دهید زیرا در صورت چندبار فراخوانی آن با خطا مواجه می‌شود.  
برای این کار می‌توانید از این قابلیت استفاده کنید.  

```csharp
using Flurl.Http;
using Flurl.Http.Newtonsoft;

namespace MyCode
{
    public class ApiServiceAgent
    {
        private readonly string _baseHttpAddress;

        static ApiServiceAgent()
        {
            FlurlHttp.Clients.UseNewtonsoft();
        }

        public ApiServiceAgent(string baseHttpAddress)
        {
            _baseHttpAddress = baseHttpAddress;
        }

        public async Task<TResponse> SendAsHttpPostAsync<TResponse, TInput>(TInput message, string url, Dictionary<string, string> headers = null)
        {
            throw new Exception(error, ex); 
        }
    }
}
```

این کانستراکتور نمی‌توانید پارامتر داشته باشد و بصورت Thread Safe فقط یکبار فراخوانی می‌شود.  
فراخوانی آن نیز بصورت خودکار انجام می‌شود و نیاز به فراخوانی دستی ندارد.  
در صورتی که در داخل این بخش خطایی داده شود این متود دیگر فراخوانی نمی‌شود.  

توضیحات بیشتر:  

[static-constructors](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/static-constructors)