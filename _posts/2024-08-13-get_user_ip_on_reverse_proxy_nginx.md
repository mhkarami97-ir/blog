---
title: "دریافت IP واقعی کاربر در زمان استفاده از Reverse Proxy در Nginx"
categories:
  - Nginx
tags:
  - nginx
  - proxy
  - ip
  - conf
---

در صورتی که از Nginx استفاده کرده باشید و در داخل آن از proxy برای انتقال درخواست‌ها به آدرس دیگر استفاده شده باشد، مقدار `X-Forwarded-For` در این حالت بطور پیش‌فرض IP سرور می‌شود نه IP کاربر فراخوانی کننده.  
بطور مثال اگر از انگولار درخواست زده شده باشد و در Backend مقدار IP ریکوست دریافت شود مقدار اشتباه می‌گیرید.  
برای حل ای مشکل باید `proxy_set_header` را به کانفیگ Nginx اضافه کنید.  

```csharp
server {
     location /api {
        rewrite ^/api/(.*)$ /$1 break;
        proxy_pass http://server.op.local:9090;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
     }
}
```

سپس در سمت Backend کافی است مقدار فوق را شبیه به کد زیر دریافت کنید.  

```csharp
namespace MyCode
{
    public class CustomActionFilterAttribute : ActionFilterAttribute
    {
        public override void OnActionExecuting(HttpActionContext context)
        {
            var ip = GetClientIp(context.Request);

            base.OnActionExecuting(context);
        }

        private string GetClientIp(HttpRequestMessage request)
        {
            try
            {
                var isSuccess = request.Headers.TryGetValues("X-Forwarded-For", out var items);

                if (isSuccess)
                {
                    return items.FirstOrDefault();
                }

                if (request.Properties.ContainsKey("MS_HttpContext"))
                {
                    return ((HttpContextWrapper)request.Properties["MS_HttpContext"]).Request.UserHostAddress;
                }

                if (request.Properties.ContainsKey(RemoteEndpointMessageProperty.Name))
                {
                    var prop = (RemoteEndpointMessageProperty)request.Properties[RemoteEndpointMessageProperty.Name];
                    return prop.Address;
                }

                return "Unknown";
            }
            catch (Exception)
            {
                return "CanNotFound";
            }
        }
    }
}
```

