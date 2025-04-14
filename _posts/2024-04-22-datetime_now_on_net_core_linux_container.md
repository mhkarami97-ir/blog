---
title: "مشکل برگشت UTC Time در DateTime.Now هاست شده در Linux"
categories:
  - Net
tags:
  - net
  - quartz
  - linux
  - container
  - datetime
---

اگر پروژه Net Core خود را بروی Docker Container از نوع Linux هاست کرده باشید، به احتمال زیاد با این مشکل مواجه می‌شوید که زمانی که در کد خود استفاده می‌کنید تایم UTC است نه تایم Local.  
بطور مثال اگر از کتابخانه quartz.net برای اجرا جاب استفاده کرده باشید و cron شما بصورت 0 0 18 * * ? باشد، انتظار دارید که هر روز ساعت 18 اجرا شود. اما اگر سرور شما تایم متفاوتی از UTC داشته باشد و بطور مثال در ایران باشد این جاب ساعت 21:30 اجرا می‌شود.  
این مشکل به دلیل تفاوت DateTime.Now در Windows , Unix می‌باشد.  

برای حل این مشکل کافی است در کد خود بصورت زیر برای دریافت تایم عمل کنید تا زمان درست برگشت داده شود:  

```csharp
var localTime = DateTime.UtcNow.ToLocalTime();
```
همچنین برای حل مشکل جاب گفته شده نیز کافی است پکیج زیر را نصب کنید:  

[Quartz.Plugins.TimeZoneConverter](https://www.nuget.org/packages/Quartz.Plugins.TimeZoneConverter)  

و سپس بصورت زیر آن را در کد خود کانفیگ کنید:  

```csharp
public static class BackgroundJob
{
    public static void ConfigureJob(this IServiceCollection services, IConfiguration configuration)
    {
        var appSettings = new AppSettings();
        configuration.GetSection("AppSettings").Bind(appSettings);

        services.AddQuartz(q =>
        {
            var job3Key = new JobKey("MyJob");

            q.AddJob<MyJobWorker>(opts => opts.WithIdentity(job3Key));

            q.AddTrigger(opts => opts
                .ForJob(job3Key)
                .WithIdentity(job3Key.Name)
                .WithCronSchedule(appSettings.MyJobCronJob));

            q.UseTimeZoneConverter();
        });

        services.AddQuartzHostedService(q => q.WaitForJobsToComplete = true);
    }
}
```

در کد بالا این بخش اضافه شده است و مهم است:  

```csharp
q.UseTimeZoneConverter()
```

در حالت استفاده دستی از کتابخانه `Quartz `

```csharp
var tehranTimeZone = TimeZoneInfo.FindSystemTimeZoneById("Asia/Tehran");

var trigger = TriggerBuilder.Create()
    .WithIdentity("ApiCallTrigger", "ApiGroup")
    .WithCronSchedule(cronExpression, x => x.InTimeZone(tehranTimeZone))
    .Build();
```

همچنین در داخل فایل `docker-compose.yml` نیز می‌توانید بخش زیر را اضافه کنید:  

```yml
services:
  myapp:
    image: image
    container_name: myapp
    environment:
      - TZ=Asia/Tehran
```

در کد بالا مقدار `TZ=Asia/Tehran` مهم است.  

