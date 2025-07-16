---
title: "استفاده Thread Safe از متود GetOrAdd در ConcurrentDictionary"
categories:
  - Net
tags:
  - net
  - concurrent_dictionary
  - thread_safe
---

در صورتی که از ConcurrentDictionary در برنامه خود استفاده می‌کنید باید به این نکته دقت کنید که متود GetOrAdd بصورت Thread Safe نیست و در صورتی که یک کلید به صورت همزمان در این متود خواسته شود ممکن است بخش Add چند بار فراخوانی شود.  
برای حل این مشکل می‌توانید از کلاس زیر در برنامه خود استفاده کنید که این مشکل را رفع می‌کند و بخش Add حداکثر یکبار به ازای کلید فراخوانی می‌شود.  
البته بیشتر از یک بار فراخوانی شدن بخش Add فقط باعث سربار این بخش می‌شود و کلید در هر صورت یکبار در لیست وجود دارد.  
استفاده از Lazy با LazyThreadSafetyMode.ExecutionAndPublication بصورت زیر باعث حل مشکل می‌شود:  
استفاده از Lazy باعث می‌شود که دسترسی به مقدار فقط وقتی به شی Value نیاز است فراخوانی شود. وقتی چند ترد به صورت همزمان یک کلید را درخواست می‌کنند برای آنها بخش Lazy ایجاد می‌شود اما با توجه به خاصیت Lazy و LazyThreadSafetyMode.ExecutionAndPublication فقط یکی از آنها بخش Add را فراخوانی می‌کنند و بقیه منتظر می‌مانند. پس بعد از اجرا مورد اول بقیه با توجه به اینکه کلید مورد نظر مقدار دارد اجرا نمی‌شوند.  

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading;

/// <summary>
/// A thread-safe concurrent dictionary wrapper that guarantees factory execution only once per key,
/// using Lazy&lt;T&gt; and ExecutionAndPublication mode.
/// </summary>
public class ThreadSafeConcurrentDictionary<TKey, TValue>
    where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, Lazy<TValue>> _concurrentDictionary;

    public ThreadSafeConcurrentDictionary()
    {
        _concurrentDictionary = new ConcurrentDictionary<TKey, Lazy<TValue>>();
    }

    /// <summary>
    /// Gets the value associated with the specified key, or adds it using the provided factory.
    /// The valueFactory is guaranteed to be invoked only once per key, even in multithreaded environments.
    /// </summary>
    public TValue GetOrAdd(TKey key, Func<TKey, TValue> valueFactory)
    {
        if (valueFactory is null)
            throw new ArgumentNullException(nameof(valueFactory));

        var lazyResult = _concurrentDictionary.GetOrAdd(
            key,
            k => new Lazy<TValue>(() => valueFactory(k), LazyThreadSafetyMode.ExecutionAndPublication)
        );

        return lazyResult.Value;
    }

    /// <summary>
    /// Clears all items from the dictionary.
    /// </summary>
    public void Clear() => _concurrentDictionary.Clear();

    /// <summary>
    /// Tries to remove the specified key from the dictionary.
    /// </summary>
    public bool TryRemove(TKey key, out TValue? value)
    {
        if (_concurrentDictionary.TryRemove(key, out var lazy))
        {
            value = lazy.Value;
            return true;
        }

        value = default;
        return false;
    }

    /// <summary>
    /// Gets the current count of items in the dictionary.
    /// </summary>
    public int Count => _concurrentDictionary.Count;

    /// <summary>
    /// Tries to get the value for the specified key.
    /// </summary>
    public bool TryGetValue(TKey key, out TValue? value)
    {
        if (_concurrentDictionary.TryGetValue(key, out var lazy))
        {
            value = lazy.Value;
            return true;
        }

        value = default;
        return false;
    }
}
```

مثال:  

```csharp
var cache = new ThreadSafeConcurrentDictionary<string, string>();

string result = cache.GetOrAdd("name", key => LoadExpensiveData(key));

string LoadExpensiveData(string key)
{
    Console.WriteLine("Expensive call running...");
    Thread.Sleep(500);
    return $"Result for {key}";
}
```