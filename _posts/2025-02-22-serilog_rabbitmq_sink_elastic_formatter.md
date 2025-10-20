---
title: "مشکل عدم ارسال بعضی از لاگ‌ها در Serilog"
categories:
  - Net
tags:
  - net
  - serilog
  - log
---

در یکی از پروژه‌ها از کدی مشابه زیر برای لاگ خطاها و نوشتن آن بر روی RabbitMQ و برداشتن آن توسط ElasticSearch استفاده می‌شد.  
در بعضی مواقع یک سری از لاگ‌ها گم می‌شد و در پنل نهایی مشاهده نمی‌شد. بعد از بررسی‌های مختلف متوجه شدیم که خطا حتی در exchange RabbitMq هم نوشته نمی‌شود.  
با توجه به اینکه سیستم APM داشت و خطاها در آن موجود بود دنبال نقطه اشتراکی بین این خطاها گشتیم که متوجه شدیم در تمام آن‌ها message موجود در Exception مقدار NULL دارد.  

```csharp
using Serilog;
using Serilog.Formatting.Elasticsearch;

namespace Utility
{
    internal class Logger : ILogger
    {
        private static readonly ILogger SerilogLogger = new LoggerConfiguration()
            .ReadFrom.AppSettings()
            .WriteTo.RabbitMQ((clientConfiguration, sinkConfiguration) =>
            {
                clientConfiguration.Username = Configs.LoggerRabbitmqUsername;
                clientConfiguration.Password = Configs.LoggerRabbitmqPassword;
                clientConfiguration.Exchange = Configs.LoggerExchange;
                clientConfiguration.ExchangeType = Configs.ExchangeTypeOfLoggerExchange;
                clientConfiguration.RouteKey = Configs.LoggerRoutingKey;
                clientConfiguration.Port = Configs.LoggerRabbitmqPort;
                sinkConfiguration.BatchPostingLimit = Configs.LoggerBatchPostingLimit;
                sinkConfiguration.TextFormatter = new ElasticsearchJsonFormatter();

                foreach (var hostname in GetHostNames())
                {
                    clientConfiguration.Hostnames.Add(hostname);
                }
            })
            .CreateLogger();

        public void Critical(Exception ex)
        {
            SerilogLogger.Fatal(ex, ex.Message);
        }

        private static List<string> GetHostNames()
        {
            var hostNames = new List<string>();
            var hosts = Configs.LoggerRabbitmqHostnames;

            if (hosts.Contains(","))
            {
                hostNames.AddRange(hosts.Split(','));
            }
            else
            {
                hostNames.Add(hosts);
            }

            return hostNames;
        }
    }
}
```

در واقع در کد بالا بخش `SerilogLogger.Fatal(ex, ex.Message)` مقدار `ex.Message` یا همان پارامتر دوم متود `Fatal` خالی بود.  
با بررسی سورس کتابخانه که در لینک زیر آمده است، برای ساخت Json نهایی با توجه به خالی بودن مقدار گفته شده کتابخانه با مشکل مواجه می‌شود و دیتا درستی ایجاد نمی‌کرد.  

[serilog-sinks-elasticsearch](https://github.com/serilog-contrib/serilog-sinks-elasticsearch/blob/dev/src/Serilog.Formatting.Elasticsearch/DefaultJsonFormatter.cs#L321)  

برای حل مشکل در لایه‌های دیگر تغییری داده شد که خطا حتما Message داشته باشد. برای اطمینان نیز کد بصورت زیر تغییر پیدا کرد تا در صورت خالی بودن مقدار پیش‌فرضی قرار داده شود.  

```csharp
public void Critical(Exception ex)
{
    SerilogLogger.Fatal(ex, ex.Message ?? "Error");
}
```