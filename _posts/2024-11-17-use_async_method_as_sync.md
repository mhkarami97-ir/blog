---
title: "استفاده از متودهای async بصورت sync"
categories:
  - Net
tags:
  - net
  - async
  - sync
  - asyncEx
  - thread
---

در صورتی که در مواقع خاص نیاز داشتید که از متودهای async بصورت sync استفاده کنید، بهترین راه استفاده بصورت زیر است.  
در این روش از `.ConfigureAwait(false).GetAwaiter().GetResult()` استفاده شده است. توسط این کار ترد شما به لاک نمی‌خورد. دقت کنید که استفاده از `ConfigureAwait(false)` مورد نیاز است در غیر این صورت در لود بالا تردهای شما با مشکل مواجه می‌شوند.  
خود استفاده از `.GetAwaiter().GetResult()` بجای استفاده از `.Result` نیز جلو DeadLock را می‌گیرد.  

دقت کنید که روش زیر در مواقع خاص است که امکان استفاده از async را ندارید. در غیر این صورت استفاده مستقیم از async پیشنهاد می‌شود.  

```csharp
public TResponse SendAsHttpGetSync<TResponse>(string url, object query = null, Dictionary<string, string> headers = null)
{
      try
      {
          return _baseHttpAddress
              .WithHeaders(headers)
              .WithTimeout(TimeSpan.FromSeconds(TimeOutOnSecond))
              .AppendPathSegment(url)
              .SetQueryParams(query)
              .GetJsonAsync<TResponse>()
              .ConfigureAwait(false).GetAwaiter().GetResult();
      }
      catch (FlurlHttpException ex)
      {
          var error = ex.GetResponseStringAsync().ConfigureAwait(false).GetAwaiter().GetResult() ?? string.Empty;

          throw new Exception(error, ex);
      }
}
```

برای راحتی می‌توانید از کتابخانه AsyncEx استفاده کنید که کار شما را راحتتر می‌کند.  
در این کتابخانه متود `AsyncContext.Run` برای راحتی کار فراهم شده است.  

```csharp
public async Task<TResponse> SendAsHttpGetAsync<TResponse>(string url, object query = null, Dictionary<string, string> headers = null)
{
      try
      {
          return await _baseHttpAddress
              .WithHeaders(headers)
              .WithTimeout(TimeSpan.FromSeconds(TimeOutOnSecond))
              .AppendPathSegment(url)
              .SetQueryParams(query)
              .GetJsonAsync<TResponse>();
      }
      catch (FlurlHttpException ex)
      {
          var error = await ex.GetResponseStringAsync() ?? string.Empty;

          throw new Exception(error, ex);
      }
}

public TResponse SendAsHttpGetSync<TResponse>()
{
    return AsyncContext.Run(SendAsHttpGetAsync);
}
```

اطلاعات بیشتر

[AsyncEx](https://github.com/StephenCleary/AsyncEx/blob/master/doc/AsyncContext.md)  
