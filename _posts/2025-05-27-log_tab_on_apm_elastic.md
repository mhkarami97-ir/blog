---
title: "نمایش Log در Elastic Apm"
categories:
  - Net
tags:
  - net
  - apm
  - log
---

در صورتی که از Elastic Apm برای Trace سیستم خود استفاده می‌کنید احتمالا تب Log را در Kibana APM دیده‌اید.  
بصورت پیش‌فرض Apm Span ها شامل Log های زده شده نیستند و برای اینکه لاگ خطاها و یا Information را هم شامل بشوند نیاز است تغییراتی در سیستم خود بدهید.  
بدین منظور کافی است کد زیر را به کانفیگ Serilog خود اضافه کنید:  

```csharp
public class OpenTelemetryEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var activity = Activity.Current;
        if (activity != null)
        {
            logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("trace.id", activity.TraceId));
            logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("parent.id", activity.ParentSpanId));
            logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("transaction.id", activity.SpanId));
        }
    }
}
```

و سپس آن را بصورت زیر استفاده کنید:  

```csharp
using System.Diagnostics;
using Serilog;
using Serilog.Core;
using Serilog.Events;
using Serilog.Filters;

namespace My.Configuration
{
    public static class Logger
    {
        public static void ConfigureLogger(this IServiceCollection builder, IConfiguration configuration)
        {
            builder.AddLogging(config =>
            {
                config.ClearProviders();

                var logger = new LoggerConfiguration()
                    .ReadFrom.Configuration(configuration)
                    .Enrich.With(new OpenTelemetryEnricher())
                    .CreateLogger();

                config.AddSerilog(logger);
            });
        }
    }
    
    public class OpenTelemetryEnricher : ILogEventEnricher
    {
        public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
        {
            var activity = Activity.Current;
            if (activity != null)
            {
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("trace.id", activity.TraceId));
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("parent.id", activity.ParentSpanId));
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("transaction.id", activity.SpanId));
            }
        }
    }
}
```

بعد از انجام این کار نیاز است Index لاگ خود را نیز مشابه عکس زیر در تعریف کنید:  

![mhkarami97](/assets/img/log-on-apm.jpg)  

با این کار هر Log شما دارای TransactionId نیز است که باعث می‌شود بصورت خودکار در Apm هم بیاید.  
دقتی کنید که برای انجام این کار نیاز است سرور Log, Apm شما یکی باشد و در صورتی که سرور آنها جدا باشد امکان مشاهده لاگ نیست.  
همچنین این روش حجم اضافه برای لاگ در Apm نمی‌گیرد و لاگ‌ها فقط یکبار در الستیک ذخیره می‌شوند.  