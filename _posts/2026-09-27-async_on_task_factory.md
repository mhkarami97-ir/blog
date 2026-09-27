---
title: "مدیریت صحیح خطا در Pipelineهای Async با BlockingCollection در .NET"
categories:
  - .NET
tags:
  - dotnet
  - csharp
  - async
  - await
  - task
  - blockingcollection
  - concurrency
  - producer-consumer
  - cancellation
---

در یک Pipeline معمولاً چند Producer داده تولید می‌کنند و یک یا چند Consumer آن‌ها را پردازش یا ذخیره می‌کنند. این الگو در ظاهر ساده است، اما ترکیب آن با `async/await`، `BlockingCollection<T>`، ظرفیت محدود، لغو عملیات و مدیریت Exception می‌تواند به توقف کامل Pipeline یا گم‌شدن خطاها منجر شود.

در این مطلب، یک خطای رایج را بررسی می‌کنیم: چند Worker برای Crawl داده تولید می‌کنند و `SaveDataAsync` هم‌زمان داده‌ها را از `BlockingCollection<CrawlData>` می‌خواند. اگر یکی از Workerها خطا کند، Consumer ممکن است تا ابد منتظر داده یا علامت پایان بماند؛ در نتیجه برنامه ظاهراً قفل می‌شود.

هدف این مقاله این است که دقیقاً بفهمیم:

- چرا `Task.Factory.StartNew` با متدهای Async خطرناک است.
- چرا نوع `List<Task<Task>>` ایجاد می‌شود.
- چرا `await Task.WhenAll` به‌تنهایی مشکل Producer-Consumer را حل نمی‌کند.
- چرا `CompleteAdding` باید در مسیر موفقیت و خطا اجرا شود.
- چگونه خطا را به Consumer منتقل و کل Pipeline را به‌صورت کنترل‌شده متوقف کنیم.
- چه زمانی `BlockingCollection` انتخاب مناسبی نیست و `Channel<T>` گزینه بهتری است.

---

## سناریوی مسئله

کد ساده‌شده‌ی Pipeline به شکل زیر است:

```csharp
private async Task RunCrawlPipelineAsync(
    List<CustomerReportHistoryInfo> customers,
    IndexDataBundle indexDataBundle,
    CancellationToken ct)
{
    var customersQueue = new ConcurrentQueue<CustomerReportHistoryInfo>(customers);

    using (var dataCollection = new BlockingCollection<CrawlData>(BoundedCapacity))
    {
        var degreeOfParallelism = GetConfigFromAppSetting(
            DopSettingKey,
            DefaultDegreeOfParallelism);

        var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
            .Select(_ => Task.Factory.StartNew(async state =>
            {
                var collection = (BlockingCollection<CrawlData>)state;

                await _customerCrawler.CrawlAsync(
                    customersQueue,
                    default,
                    collection,
                    ct);
            },
            dataCollection,
            ct,
            TaskCreationOptions.LongRunning,
            TaskScheduler.Default))
            .ToList();

        var saveTask = _crawlDataPersister.SaveDataAsync(
            dataCollection,
            indexDataBundle.Index,
            indexDataBundle.WeightedIndex,
            ct);

        await Task.WhenAll(crawlTasks);

        dataCollection.CompleteAdding();

        await saveTask;
    }
}
```

ایده این است:

```text
customers
    |
    v
ConcurrentQueue<Customer>
    |
    +--> Crawl Worker 1 --+
    +--> Crawl Worker 2 --+----> BlockingCollection<CrawlData> ----> SaveDataAsync
    +--> Crawl Worker 3 --+
```

در این مدل، Crawl Workerها Producer هستند و `SaveDataAsync` Consumer است.

---

## مشکل اول: StartNew و async

کد زیر از نظر ظاهری طبیعی است:

```csharp
Task.Factory.StartNew(async () =>
{
    await DoWorkAsync();
});
```

اما نوع واقعی خروجی این عبارت `Task<Task>` است.

دلیل آن این است که `StartNew` یک Delegate را اجرا می‌کند و Delegate شما به دلیل وجود `async`، خودش یک `Task` برمی‌گرداند:

```text
StartNew
  Outer Task
      Inner Task returned by async delegate
```

Task بیرونی فقط اجرای اولیه Delegate را نمایش می‌دهد. وقتی اجرای Delegate به اولین `await` برسد، Task بیرونی ممکن است Complete شود؛ درحالی‌که Task داخلی هنوز در حال اجراست.

در نتیجه این کد:

```csharp
var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
    .Select(_ => Task.Factory.StartNew(async () =>
    {
        await _customerCrawler.CrawlAsync(...);
    }))
    .ToList();
```

نوع زیر را تولید می‌کند:

```csharp
List<Task<Task>>
```

پس این خط:

```csharp
await Task.WhenAll(crawlTasks);
```

ممکن است فقط Taskهای بیرونی را منتظر بماند، نه عملیات واقعی Crawl را.

### نتیجه این خطا

- `Task.WhenAll` زودتر از پایان Crawl تمام می‌شود.
- `CompleteAdding` ممکن است زود اجرا شود.
- Exception واقعی داخل Task داخلی باقی می‌ماند.
- Exception ممکن است هیچ‌وقت توسط کد شما مشاهده نشود.
- لاگ‌هایی که بعد از یک `await` قرار دارند ممکن است اصلاً اجرا نشوند یا در زمان نامناسب اجرا شوند.

مایکروسافت برای async lambdaها و تفاوت `StartNew` با `Task.Run` همین خطر را مستند کرده است. `Task.Run` برای Delegateهای async، Task تو‌در‌تو را Unwrap می‌کند؛ اما `StartNew` این کار را خودکار انجام نمی‌دهد.

---

## اصلاح مشکل StartNew

اگر واقعاً به `Task.Factory.StartNew` نیاز ندارید، بهترین راه حذف آن است:

```csharp
var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
    .Select(_ => _customerCrawler.CrawlAsync(
        customersQueue,
        ct,
        dataCollection,
        ct))
    .ToList();
```

در این حالت، اگر `CrawlAsync` از نوع `Task` باشد، متغیر `crawlTasks` از نوع زیر خواهد بود:

```csharp
List<Task>
```

سپس:

```csharp
await Task.WhenAll(crawlTasks);
```

واقعاً تا پایان تمام Workerها صبر می‌کند.

اگر به اجرای Thread جداگانه نیاز واقعی دارید، باید Task داخلی را Unwrap کنید:

```csharp
var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
    .Select(_ => Task.Factory.StartNew(
            () => _customerCrawler.CrawlAsync(
                customersQueue,
                ct,
                dataCollection,
                ct),
            ct,
            TaskCreationOptions.LongRunning,
            TaskScheduler.Default)
        .Unwrap())
    .ToList();
```

بااین‌حال، برای متدهای واقعاً Async معمولاً `LongRunning` انتخاب مناسبی نیست. این گزینه برای کارهای طولانی و CPU-bound یا کارهایی که یک Thread را به‌صورت پیوسته اشغال می‌کنند مفید است. در یک عملیات I/O-bound که درست با `await` نوشته شده، Thread هنگام انتظار آزاد می‌شود و اختصاص Thread جداگانه معمولاً سودی ندارد.

---

## مشکل اصلی: Consumer منتظر CompleteAdding می‌ماند

حالا فرض کنیم مشکل `Task<Task>` را حل کرده‌ایم. همچنان این ساختار خطرناک است:

```csharp
await Task.WhenAll(crawlTasks);
dataCollection.CompleteAdding();
await saveTask;
```

فرض کنید `SaveDataAsync` شبیه این باشد:

```csharp
public async Task SaveDataAsync(
    BlockingCollection<CrawlData> dataCollection,
    Index index,
    WeightedIndex weightedIndex,
    CancellationToken ct)
{
    foreach (var data in dataCollection.GetConsumingEnumerable(ct))
    {
        await SaveItemAsync(data, index, weightedIndex, ct);
    }
}
```

`GetConsumingEnumerable` تا زمانی ادامه پیدا می‌کند که:

1. همه آیتم‌های موجود مصرف شوند.
2. تولیدکننده‌ها اعلام کنند که دیگر آیتمی اضافه نمی‌شود.

اعلام پایان تولید با این دستور انجام می‌شود:

```csharp
dataCollection.CompleteAdding();
```

اگر یکی از Crawl Workerها Exception بدهد، این خط هیچ‌وقت اجرا نمی‌شود:

```csharp
await Task.WhenAll(crawlTasks);
```

چون `await` با Exception متوقف می‌شود و اجرای متد به خط بعد نمی‌رسد. بنابراین:

```csharp
dataCollection.CompleteAdding();
```

اجرا نمی‌شود.

از طرف دیگر، `SaveDataAsync` همچنان در حال خواندن از `BlockingCollection` است. اگر Collection خالی باشد، Consumer منتظر می‌ماند که آیتم جدید بیاید یا Collection Complete شود. اما چون Producer شکست خورده و `CompleteAdding` اجرا نشده، Consumer نمی‌فهمد که دیگر داده‌ای تولید نخواهد شد.

چرخه به این شکل درمی‌آید:

```text
Crawl Worker خطا می‌دهد
        |
        v
Task.WhenAll خطا می‌دهد
        |
        v
CompleteAdding اجرا نمی‌شود
        |
        v
SaveDataAsync روی Collection خالی منتظر می‌ماند
        |
        v
Pipeline ظاهراً قفل می‌شود
```

این معمولاً Deadlock کلاسیک به معنای وجود دو Lock نیست؛ بلکه یک انتظار بی‌نهایت در Producer-Consumer است. Consumer منتظر Signal پایان است و Signal پایان به‌دلیل Exception Producer هرگز ارسال نمی‌شود.

طبق مستندات `BlockingCollection`، Producer باید پس از پایان تولید `CompleteAdding` را فراخوانی کند و Consumer می‌تواند با `GetConsumingEnumerable` تا خالی‌شدن Collection و پایان تولید ادامه دهد.

---

## چرا اجرای SaveDataAsync روی Thread جدا راه‌حل واقعی نیست

برای حل مشکل، ممکن است این کار انجام شود:

```csharp
var saveTask = Task.Run(() => _crawlDataPersister.SaveDataAsync(
    dataCollection,
    indexDataBundle.Index,
    indexDataBundle.WeightedIndex,
    ct));
```

این کد ممکن است باعث شود Thread اصلی بلافاصله آزاد شود، اما مشکل اصلی را حل نمی‌کند.

مشکل واقعی این نیست که `SaveDataAsync` روی Thread اصلی اجرا شده است. یک متد Async وقتی به `await` می‌رسد، Thread را بلوکه نمی‌کند. مشکل اصلی این است که Producer در مسیر خطا `CompleteAdding` را اجرا نمی‌کند.

بنابراین اجرای Consumer روی Thread جدا فقط مشکل را پنهان می‌کند:

- Consumer روی Thread یا Task دیگری همچنان منتظر می‌ماند.
- Exception Producer هنوز باید مدیریت شود.
- ممکن است منابع Dispose شوند، درحالی‌که Consumer هنوز از Collection می‌خواند.
- ممکن است Task مربوط به ذخیره‌سازی بدون مشاهده باقی بماند.
- برنامه در Shutdown یا Cancellation رفتار نامشخص پیدا می‌کند.

راه‌حل باید روی Lifecycle صحیح Collection و هماهنگی Taskها تمرکز کند، نه صرفاً روی انتقال Consumer به Thread دیگر.

مایکروسافت نیز توصیه می‌کند برای عملیات Async از زنجیره `async/await` استفاده شود و به‌جای Block کردن Thread، Taskها با `await Task.WhenAll` ترکیب شوند.

---

## اصل مهم: CompleteAdding باید در finally باشد

حداقل اصلاح ضروری این است:

```csharp
try
{
    await Task.WhenAll(crawlTasks);
}
finally
{
    dataCollection.CompleteAdding();
}

await saveTask;
```

در این ساختار، چه Workerها موفق شوند و چه یکی از آن‌ها خطا بدهد، Collection در هر صورت Complete می‌شود.

اما این نسخه هنوز یک مشکل طراحی دارد: اگر Crawl Worker شکست بخورد، `SaveDataAsync` ممکن است داده‌های ناقص را ذخیره کند. بنابراین باید تصمیم بگیریم که رفتار موردنظر چیست:

- آیا با خطای یک Worker باید بقیه Workerها نیز متوقف شوند؟
- آیا باید داده‌های موفق تا آن لحظه ذخیره شوند؟
- آیا Pipeline باید به‌صورت Partial Success تمام شود؟
- آیا باید Exception اصلی به Caller منتقل شود؟

برای بیشتر Pipelineهای داده‌ای، رفتار قابل‌اعتماد این است که با خطای جدی یک Producer، Cancellation داخلی فعال شود، همه Producerها و Consumerها متوقف شوند، Collection Complete شود و Exception اصلی دوباره به Caller برسد.

---

## پیاده‌سازی پیشنهادی با Cancellation داخلی

کد زیر یک الگوی کامل‌تر ارائه می‌کند:

```csharp
private async Task RunCrawlPipelineAsync(
    List<CustomerReportHistoryInfo> customers,
    IndexDataBundle indexDataBundle,
    CancellationToken ct)
{
    var customersQueue = new ConcurrentQueue<CustomerReportHistoryInfo>(customers);
    var degreeOfParallelism = GetDegreeOfParallelism();

    using var dataCollection = new BlockingCollection<CrawlData>(BoundedCapacity);
    using var pipelineCts = CancellationTokenSource.CreateLinkedTokenSource(ct);

    var pipelineToken = pipelineCts.Token;

    var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
        .Select(_ => RunCrawlerWorkerAsync(
            customersQueue,
            dataCollection,
            pipelineCts,
            pipelineToken))
        .ToArray();

    var saveTask = _crawlDataPersister.SaveDataAsync(
        dataCollection,
        indexDataBundle.Index,
        indexDataBundle.WeightedIndex,
        pipelineToken);

    Exception? pipelineException = null;

    try
    {
        await Task.WhenAll(crawlTasks).ConfigureAwait(false);
    }
    catch (Exception exception)
    {
        pipelineException = exception;
        pipelineCts.Cancel();

        Log.Error(
            exception,
            "Crawl pipeline failed while one or more crawler workers were running.");
    }
    finally
    {
        dataCollection.CompleteAdding();
    }

    try
    {
        await saveTask.ConfigureAwait(false);
    }
    catch (OperationCanceledException) when (pipelineToken.IsCancellationRequested)
    {
        Log.Warning("Save operation was canceled because the crawl pipeline stopped.");
    }
    catch (Exception exception)
    {
        Log.Error(exception, "Crawl data persistence failed.");
        pipelineException ??= exception;
    }

    if (pipelineException is not null)
    {
        throw pipelineException;
    }
}
```

Worker مربوط به Crawl:

```csharp
private async Task RunCrawlerWorkerAsync(
    ConcurrentQueue<CustomerReportHistoryInfo> customersQueue,
    BlockingCollection<CrawlData> dataCollection,
    CancellationTokenSource pipelineCts,
    CancellationToken pipelineToken)
{
    try
    {
        await _customerCrawler.CrawlAsync(
            customersQueue,
            pipelineToken,
            dataCollection,
            pipelineToken)
            .ConfigureAwait(false);
    }
    catch (OperationCanceledException) when (pipelineToken.IsCancellationRequested)
    {
        Log.Warning("Crawler worker was canceled.");
        throw;
    }
    catch (Exception exception)
    {
        Log.Error(exception, "Crawler worker failed.");
        pipelineCts.Cancel();
        throw;
    }
}
```

متد خواندن تنظیمات:

```csharp
private int GetDegreeOfParallelism()
{
    var degreeOfParallelism = GetConfigFromAppSetting(
        DopSettingKey,
        DefaultDegreeOfParallelism);

    if (degreeOfParallelism >= 1)
    {
        return degreeOfParallelism;
    }

    Log.Warning(
        "Configured {SettingKey} value {Value} is invalid, falling back to {Default}",
        DopSettingKey,
        degreeOfParallelism,
        DefaultDegreeOfParallelism);

    return DefaultDegreeOfParallelism;
}
```

### نکته درباره این پیاده‌سازی

در این نسخه، `CompleteAdding` در `finally` اجرا می‌شود؛ بنابراین Consumer در مسیر خطا برای همیشه منتظر نمی‌ماند.

همچنین با `CreateLinkedTokenSource` دو نوع Cancellation به هم متصل می‌شوند:

- Cancellation خارجی که از Caller می‌آید.
- Cancellation داخلی که با خطای یکی از Workerها فعال می‌شود.

اگر Caller عملیات را لغو کند، تمام بخش‌های Pipeline آن را می‌بینند. اگر یکی از Workerها خطا کند، `pipelineCts.Cancel()` باعث می‌شود Workerها و Consumerهای دیگر نیز فرصت خروج کنترل‌شده داشته باشند.

---

## یک نکته مهم درباره Catch کردن Exception

استفاده از این کد:

```csharp
try
{
    await Task.WhenAll(crawlTasks);
}
catch (Exception exception)
{
    Log.Error(exception, "Crawl failed.");
}
```

به‌تنهایی کافی نیست؛ چون بعد از Catch ممکن است `saveTask` هنوز فعال باشد و از Collection بخواند. باید اول Collection را Complete کنید و سپس Consumer را نیز منتظر بمانید.

الگوی صحیح Lifecycle این است:

```text
Start Producers
Start Consumer

Wait for Producers
    |
    +-- Success --> CompleteAdding
    |
    +-- Failure --> Cancel pipeline + CompleteAdding

Wait for Consumer

Propagate final exception
Dispose resources
```

نباید Collection را قبل از پایان Producerها Complete کنید؛ چون Producerها ممکن است هنگام `Add` با `InvalidOperationException` مواجه شوند.

همچنین نباید قبل از پایان Consumer، `BlockingCollection` را Dispose کنید؛ چون Consumer هنوز ممکن است در حال `Take` یا `GetConsumingEnumerable` باشد.

---

## نسخه ساده‌تر برای اغلب پروژه‌ها

اگر نمی‌خواهید Exceptionها را پیچیده تجمیع کنید، نسخه زیر برای بسیاری از سرویس‌ها مناسب است:

```csharp
private async Task RunCrawlPipelineAsync(
    List<CustomerReportHistoryInfo> customers,
    IndexDataBundle indexDataBundle,
    CancellationToken ct)
{
    var customersQueue = new ConcurrentQueue<CustomerReportHistoryInfo>(customers);
    var degreeOfParallelism = GetDegreeOfParallelism();

    using var dataCollection = new BlockingCollection<CrawlData>(BoundedCapacity);
    using var pipelineCts = CancellationTokenSource.CreateLinkedTokenSource(ct);

    var pipelineToken = pipelineCts.Token;

    var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
        .Select(_ => _customerCrawler.CrawlAsync(
            customersQueue,
            pipelineToken,
            dataCollection,
            pipelineToken))
        .ToArray();

    var saveTask = _crawlDataPersister.SaveDataAsync(
        dataCollection,
        indexDataBundle.Index,
        indexDataBundle.WeightedIndex,
        pipelineToken);

    try
    {
        await Task.WhenAll(crawlTasks).ConfigureAwait(false);
    }
    catch
    {
        pipelineCts.Cancel();
        throw;
    }
    finally
    {
        dataCollection.CompleteAdding();
    }

    await saveTask.ConfigureAwait(false);
}
```

این نسخه برای حالتی خوب است که:

- با خطای Crawl باید کل Pipeline Fail شود.
- `SaveDataAsync` با CancellationToken به‌درستی خارج می‌شود.
- نیاز ندارید Exception ذخیره‌سازی را با Exception Crawl ادغام کنید.

اما یک نکته وجود دارد: در این نسخه اگر `Task.WhenAll` خطا دهد، به‌دلیل `throw`، اجرای کد به `await saveTask` نمی‌رسد. چون `finally` فقط `CompleteAdding` را اجرا می‌کند. اگر Consumer نیاز دارد حتماً await شود، باید آن را در ساختار جامع‌تر قبلی مدیریت کنید.

---

## نسخه مقاوم‌تر با انتظار هم‌زمان Consumer و Producer

برای اینکه در هیچ مسیری Task ذخیره‌سازی رها نشود، می‌توان از `Task.WhenAll` روی Consumer و Producerها استفاده کرد؛ اما ترتیب Complete کردن همچنان ضروری است:

```csharp
private async Task RunCrawlPipelineAsync(
    List<CustomerReportHistoryInfo> customers,
    IndexDataBundle indexDataBundle,
    CancellationToken ct)
{
    var customersQueue = new ConcurrentQueue<CustomerReportHistoryInfo>(customers);
    var degreeOfParallelism = GetDegreeOfParallelism();

    using var dataCollection = new BlockingCollection<CrawlData>(BoundedCapacity);
    using var pipelineCts = CancellationTokenSource.CreateLinkedTokenSource(ct);

    var pipelineToken = pipelineCts.Token;

    var crawlTasks = Enumerable.Range(0, degreeOfParallelism)
        .Select(_ => RunCrawlerWorkerAsync(
            customersQueue,
            dataCollection,
            pipelineCts,
            pipelineToken))
        .ToArray();

    var saveTask = _crawlDataPersister.SaveDataAsync(
        dataCollection,
        indexDataBundle.Index,
        indexDataBundle.WeightedIndex,
        pipelineToken);

    Exception? firstException = null;

    try
    {
        try
        {
            await Task.WhenAll(crawlTasks).ConfigureAwait(false);
        }
        catch (Exception exception)
        {
            firstException = exception;
            pipelineCts.Cancel();
        }
    }
    finally
    {
        dataCollection.CompleteAdding();
    }

    try
    {
        await saveTask.ConfigureAwait(false);
    }
    catch (Exception exception) when (firstException is not null)
    {
        Log.Error(
            exception,
            "Save operation also failed after crawler pipeline failure.");
    }

    if (firstException is not null)
    {
        throw firstException;
    }
}
```

این الگو دو ویژگی مهم دارد:

- Consumer هرگز رها نمی‌شود.
- خطای Producer باعث می‌شود Producerهای دیگر و Consumer فرصت خروج داشته باشند.

---

## آیا باید SaveDataAsync را روی Thread جدا اجرا کنیم؟

معمولاً خیر.

اگر `SaveDataAsync` واقعاً Async باشد و عملیات I/O را با APIهای Async انجام دهد، این کد کافی است:

```csharp
var saveTask = _crawlDataPersister.SaveDataAsync(
    dataCollection,
    indexDataBundle.Index,
    indexDataBundle.WeightedIndex,
    pipelineToken);
```

فراخوانی متد Async تا اولین نقطه توقف اجرا می‌شود و سپس یک Task برمی‌گرداند. `await` روی آن Thread را Blocking نمی‌کند.

استفاده از `Task.Run` برای متد Async معمولاً فقط زمانی توجیه دارد که بخشی از کار واقعاً CPU-bound یا کاملاً synchronous باشد:

```csharp
var saveTask = Task.Run(
    () => _crawlDataPersister.SaveDataAsync(
        dataCollection,
        indexDataBundle.Index,
        indexDataBundle.WeightedIndex,
        pipelineToken),
    pipelineToken);
```

اما حتی در این حالت هم `Task.Run` مشکل Signal پایان Collection، Cancellation یا Exception را حل نمی‌کند. فقط محل شروع اجرای متد را تغییر می‌دهد.

قاعده عملی:

```text
I/O-bound و Async واقعی  --> صدا زدن مستقیم متد Async
CPU-bound                --> بررسی Task.Run با اندازه‌گیری واقعی
Sync طولانی              --> بررسی Thread اختصاصی یا معماری جداگانه
```

---

## BlockingCollection و ظرفیت محدود

در کد شما Collection دارای ظرفیت محدود است:

```csharp
var dataCollection = new BlockingCollection<CrawlData>(boundedCapacity: BoundedCapacity);
```

این کار برای کنترل فشار حافظه مفید است. اگر Consumer کند باشد و Collection پر شود، Producer هنگام Add منتظر می‌ماند تا Consumer آیتمی بردارد.

این رفتار یک Backpressure طبیعی ایجاد می‌کند:

```text
Producer سریع‌تر از Consumer
        |
        v
Collection پر می‌شود
        |
        v
Producer موقتاً متوقف می‌شود
        |
        v
Consumer داده ذخیره می‌کند
        |
        v
ظرفیت آزاد می‌شود
```

اما همین قابلیت می‌تواند مسیر خطا را پیچیده کند. اگر Consumer به دلیل Exception متوقف شود و Producer همچنان `Add` کند، Collection پر می‌شود و Producerها نیز متوقف می‌شوند.

بنابراین `SaveDataAsync` هم باید Exception خود را کنترل کند و در صورت شکست، Cancellation داخلی را فعال کند تا Producerها در `Add` بی‌نهایت منتظر نمانند.

مثلاً در Consumer:

```csharp
private async Task SaveDataSafelyAsync(
    BlockingCollection<CrawlData> dataCollection,
    Index index,
    WeightedIndex weightedIndex,
    CancellationTokenSource pipelineCts,
    CancellationToken pipelineToken)
{
    try
    {
        foreach (var data in dataCollection.GetConsumingEnumerable(pipelineToken))
        {
            await SaveItemAsync(
                data,
                index,
                weightedIndex,
                pipelineToken)
                .ConfigureAwait(false);
        }
    }
    catch (Exception exception)
    {
        Log.Error(exception, "Data persistence consumer failed.");
        pipelineCts.Cancel();
        throw;
    }
}
```

بدون Cancellation، خطای Consumer می‌تواند Producerها را پشت Collection پرشده متوقف کند.

---

## خطاهای رایج در این معماری

### ۱. فراخوانی CompleteAdding فقط در مسیر موفقیت

```csharp
await Task.WhenAll(crawlTasks);
dataCollection.CompleteAdding();
```

این کد در صورت خطا، Signal پایان را ارسال نمی‌کند.

نسخه درست:

```csharp
try
{
    await Task.WhenAll(crawlTasks);
}
finally
{
    dataCollection.CompleteAdding();
}
```

### ۲. استفاده از StartNew با async lambda بدون Unwrap

```csharp
Task.Factory.StartNew(async () => await DoWorkAsync());
```

این عبارت `Task<Task>` ایجاد می‌کند.

نسخه بهتر:

```csharp
Task.Run(() => DoWorkAsync());
```

یا اگر StartNew ضروری است:

```csharp
Task.Factory.StartNew(
    () => DoWorkAsync(),
    cancellationToken,
    TaskCreationOptions.LongRunning,
    TaskScheduler.Default)
.Unwrap();
```

### ۳. استفاده از `default` به‌جای CancellationToken واقعی

در کد اولیه این بخش وجود داشت:

```csharp
await _customerCrawler.CrawlAsync(
    customersQueue,
    default,
    collection,
    ct);
```

اگر پارامتر دوم نیز `CancellationToken` است، ارسال `default` باعث می‌شود آن بخش از متد به Cancellation اصلی واکنش نشان ندهد.

باید بررسی شود که هر Token برای چه منظوری است. اگر هر دو باید به یک لغو مشترک پاسخ دهند:

```csharp
await _customerCrawler.CrawlAsync(
    customersQueue,
    pipelineToken,
    collection,
    pipelineToken);
```

### ۴. Dispose کردن Collection قبل از پایان Consumer

```csharp
using var dataCollection = new BlockingCollection<CrawlData>();

var saveTask = SaveDataAsync(dataCollection);

return;
```

با خروج از Scope، Collection Dispose می‌شود؛ درحالی‌که `saveTask` هنوز فعال است. همیشه باید Consumer را await کنید:

```csharp
await saveTask;
```

و بعد اجازه دهید Scope بسته شود.

### ۵. خوردن Exception و ادامه دادن بدون تصمیم

```csharp
catch (Exception exception)
{
    Log.Error(exception, "Failed");
}
```

اگر فقط لاگ بگیرید و Cancellation نکنید، سایر بخش‌ها ممکن است در وضعیت ناقص ادامه دهند. بعد از لاگ باید مشخص کنید Pipeline باید Fail، Cancel یا Partial Success شود.

---

## آیا Channel<T> انتخاب بهتری است؟

اگر کد شما کاملاً Async است، `System.Threading.Channels.Channel<T>` معمولاً از `BlockingCollection<T>` مناسب‌تر است.

`BlockingCollection` APIهای blocking مانند `Add` و `Take` دارد و برای Thread-based Producer-Consumer بسیار مناسب است. اما `Channel<T>` از ابتدا برای انتقال Async داده بین Producer و Consumer طراحی شده و APIهایی مانند `WriteAsync`، `ReadAsync` و `WaitToReadAsync` ارائه می‌دهد.

نمونه تعریف Channel با ظرفیت محدود:

```csharp
var channel = Channel.CreateBounded<CrawlData>(
    new BoundedChannelOptions(BoundedCapacity)
    {
        FullMode = BoundedChannelFullMode.Wait,
        SingleReader = true,
        SingleWriter = false
    });
```

Producer:

```csharp
private async Task RunCrawlerWorkerAsync(
    ConcurrentQueue<CustomerReportHistoryInfo> customersQueue,
    ChannelWriter<CrawlData> writer,
    CancellationToken ct)
{
    await _customerCrawler.CrawlAsync(
        customersQueue,
        ct,
        writer,
        ct);
}
```

Consumer:

```csharp
private async Task SaveDataAsync(
    ChannelReader<CrawlData> reader,
    Index index,
    WeightedIndex weightedIndex,
    CancellationToken ct)
{
    await foreach (var data in reader.ReadAllAsync(ct))
    {
        await SaveItemAsync(data, index, weightedIndex, ct);
    }
}
```

هماهنگ‌سازی پایان Producerها:

```csharp
try
{
    await Task.WhenAll(crawlTasks);
}
finally
{
    channel.Writer.TryComplete();
}
```

در صورت Exception نیز می‌توانید خطا را به Channel منتقل کنید:

```csharp
channel.Writer.TryComplete(exception);
```

این کار به Consumer می‌گوید که Channel با خطا بسته شده است.

برای Pipelineهای جدید و Async-first، `Channel<T>` اغلب API شفاف‌تری دارد. بااین‌حال اگر پروژه شما بر پایه `BlockingCollection` است و عملیات فعلی درست مدیریت شود، الزاماً نیازی به بازنویسی فوری ندارید.

---

## الگوی تصمیم‌گیری پیشنهادی

برای این مسئله می‌توان این قواعد را به‌عنوان Checklist استفاده کرد:

- اگر متد شما Async است، آن را مستقیم فراخوانی کنید و از `StartNew` پرهیز کنید.
- اگر `StartNew` با async lambda استفاده می‌شود، حتماً `.Unwrap()` را بررسی کنید.
- اگر Producer-Consumer دارید، پایان Producerها باید در مسیر موفقیت و خطا Signal شود.
- `CompleteAdding` را در `finally` قرار دهید.
- برای توقف زنجیره از CancellationToken مشترک یا Linked CancellationToken استفاده کنید.
- اگر Consumer خطا کرد، Producerها نباید تا ابد روی Collection پرشده منتظر بمانند.
- قبل از Dispose کردن Collection، تمام Producerها و Consumerها را await کنید.
- Exception را فقط Log نکنید؛ درباره Fail، Cancel یا Partial Success تصمیم بگیرید.
- برای کدهای Async-first، استفاده از `Channel<T>` را بررسی کنید.
- برای تصمیم درباره `Task.Run` یا `LongRunning`، از اندازه‌گیری واقعی استفاده کنید، نه حدس.

---

## جمع‌بندی

مشکل اصلی این Pipeline اجرای `SaveDataAsync` روی Thread اصلی نبود. مشکل واقعی در هماهنگی پایان Producerها و Consumer بود.

وقتی یکی از `crawlTasks` خطا می‌خورد، اجرای `await Task.WhenAll(crawlTasks)` متوقف می‌شود و کدی که بعد از آن قرار دارد اجرا نمی‌شود. در نتیجه `dataCollection.CompleteAdding()` فراخوانی نمی‌شود. از آنجا که `SaveDataAsync` با `GetConsumingEnumerable` منتظر پایان Collection است، در حالت خالی یا نیمه‌تمام برای همیشه منتظر می‌ماند.

اجرای Consumer روی Thread جدا فقط این انتظار را به Task دیگری منتقل می‌کند. راه‌حل اصولی شامل این موارد است:

```text
متد Async واقعی
      + await صحیح
      + Taskهای Unwrap‌شده
      + Cancellation مشترک
      + CompleteAdding در finally
      + await کردن Consumer
      + Dispose پس از پایان همه Taskها
```

کد کلیدی که باید همیشه در ذهن بماند:

```csharp
try
{
    await Task.WhenAll(producerTasks);
}
finally
{
    collection.CompleteAdding();
}

await consumerTask;
```

این الگو تضمین می‌کند Consumer چه در مسیر موفقیت و چه در مسیر خطا، Signal پایان دریافت کند و Pipeline به انتظار بی‌نهایت وارد نشود.