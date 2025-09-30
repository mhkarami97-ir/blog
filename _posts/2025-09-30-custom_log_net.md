---
title: "استفاده از Custom Log در Net"
categories:
  - Net
tags:
  - net
  - log
  - extension
---

اگه نیاز دارید که متود دلخواه خودتون برای لاگ زدن رو داشته باشید و یا بطور مثال زمان زدن لاگ خطا یک عملیات دیگه مثل Trace هم انجام بدید می‌تونید از Extension Method استفاده کنید و کارکرد ILogger رو افزایش بدید.  
برای این کار کافیه متودی شبیه به زیر بنویسید:  

```
public static class LoggerExtensions
{
    public static void LogCustomError(this ILogger logger, Exception ex, string message)
    {
        logger.LogError(exception: ex, message: "{MessageError}", message);
        Activity.Current?.RecordException(ex);
    }
}
```

و بعدش جاهی مختلف برنامه بهش دسترسی دارید و می‌تونید ازش استفاده کنید. نمونه استفاده:  

```
try {

}
catch (Exception ex) {
    _logger.LogCustomError(ex, "My Message");
}
```