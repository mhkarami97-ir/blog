---
title: "استفاده از SoftLimit بجای RateLimit در API"
categories:
  - Net
tags:
  - net
  - limit
  - ddos
---

از موارد مهمی که در ارائه API باید حواسمان به آن باشد بحث جلوگیری از فراخوانی زیاد توسط یک فرد زیاد است تا جلوی مواردی مانند حمله DDOS گرفته شود.  
یکی از راهکارها استفاده از کتابخانه‌ای مانند AspNetCoreRateLimit است تا یک کاربر را محدود کرد. راه دیگر استفاده از SoftLimit است. به این صورت که بجای خطا دادن در صورت بلاک شدن کاربر، پاسخ کاربر را می‌دهیم ولی با تایم بیشتر. بطور مثال اگر در 1 ثانیه بیشتر از 10 درخواست داشت بجای سریع جواب دادن پاسخ او را با 5 ثانیه تاخیر می‌دهیم.  
برای پیاده‌سازی این مورد می‌توانید از Middleware زیر استفاده کنید:  

```csharp
public class SoftRateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SoftRateLimitOptions _options;
    private readonly ConcurrentDictionary<string, SlidingCounter> _counters = new();
    private readonly Random _random = new();

    public SoftRateLimitMiddleware(RequestDelegate next, IOptions<SoftRateLimitOptions> options)
    {
        _next = next;
        _options = options.Value;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var ip = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var counter = _counters.GetOrAdd(ip, _ => new SlidingCounter());

        counter.Increment();

        var recentRequests = counter.RequestsInLastSecond();
        if (recentRequests > _options.RequestsPerSecond)
        {
            var baseDelay = _random.Next(_options.MinDelayMs, _options.MaxDelayMs);

            if (recentRequests > _options.AggressiveThreshold)
            {
                // Scale delay up to 2x for very aggressive traffic
                var scale = Math.Min(2.0, (double)recentRequests / _options.AggressiveThreshold);
                baseDelay = (int)(baseDelay * scale);
            }

            await Task.Delay(baseDelay);
        }

        await _next(context);
    }

    private class SlidingCounter
    {
        private readonly ConcurrentQueue<DateTime> _timestamps = new();

        public void Increment()
        {
            var now = DateTime.UtcNow;
            _timestamps.Enqueue(now);

            while (_timestamps.TryPeek(out var ts) && ts < now.AddSeconds(-1))
                _timestamps.TryDequeue(out _);
        }

        public int RequestsInLastSecond()
        {
            var cutoff = DateTime.UtcNow.AddSeconds(-1);
            return _timestamps.Count(ts => ts >= cutoff);
        }
    }
}

```

```csharp
public class SoftRateLimitOptions
{
    public int RequestsPerSecond { get; set; }
    public int AggressiveThreshold { get; set; }
    public int MinDelayMs { get; set; }
    public int MaxDelayMs { get; set; }
}

```

```csharp
{
  "SoftRateLimit": {
    "RequestsPerSecond": 10,
    "AggressiveThreshold": 100,
    "MinDelayMs": 5000,
    "MaxDelayMs": 20000
  }
}
```

و سپس بصورت زیر از آن استفاده کنید. دقت کنید که محل نوشتن کد زیر مهم است تا این اعتبارسنجی قبل از موارد دیگر مثل احراز هویت و یا فراخوانی API اصلی شما که شامل مواردی مانند فراخوانی دیتابیس است باشد.  

```csharp
app.UseRouting();

// Soft limit should come BEFORE:
app.UseMiddleware<SoftRateLimitMiddleware>();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```