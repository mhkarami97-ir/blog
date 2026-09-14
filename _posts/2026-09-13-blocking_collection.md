---
title: "چه زمانی از BlockingCollection و چه زمانی از Channel استفاده کنیم؟"
categories:
  - Net
tags:
  - net
  - collection
  - queue
---

## بخش ۱: مسئله چیست و BlockingCollection چرا به وجود آمد

فرض کنید برنامه‌ای دارید که در آن چند Thread تولیدکننده داده (Producer) و چند Thread مصرف‌کننده (Consumer) وجود دارند که باید آن داده‌ها را پردازش کنند. مثال کلاسیک: یک Thread فایل‌های لاگ را می‌خوانَد و در صف قرار می‌دهد، Thread دیگری آن‌ها را پردازش و در دیتابیس ذخیره می‌کند.

اگر این کار با یک `Queue<T>` معمولی پیاده‌سازی شود، دو مشکل اساسی بروز می‌کند:

اول، `Queue<T>` ذاتاً Thread-safe نیست. اگر دو Thread همزمان به آن `Enqueue`/`Dequeue` بزنند، ساختار داخلی صف (که بر پایه آرایه است) خراب می‌شود. راه‌حل اولیه، قفل‌گذاری دستی با `lock` است، اما این کار خودش دو ریسک به همراه دارد: افت شدید Performance به‌خاطر Contention روی قفل، و احتمال Deadlock در صورت مدیریت نادرست.

دوم، حتی اگر مشکل Thread-safety با `ConcurrentQueue<T>` حل شود، مشکل دیگری باقی می‌ماند: وقتی صف خالی است، Consumer باید چه کار کند؟ اگر پیوسته در یک حلقه `while` بررسی شود که آیا آیتم جدیدی رسیده یا نه (Polling/Busy-Waiting)، منابع CPU بی‌دلیل مصرف می‌شود. اگر هم منطق «صبر تا رسیدن داده» با `ManualResetEvent` یا `Monitor.Wait/Pulse` به‌صورت دستی پیاده‌سازی شود، کد پیچیده و مستعد باگ‌های Race Condition خواهد شد.

`BlockingCollection<T>` دقیقاً برای حل همین دو مسئله در .NET Framework 4.0 معرفی شد: پیاده‌سازی آماده و تست‌شده الگوی Producer-Consumer، به‌همراه قابلیت Blocking خودکار (Thread واقعاً می‌خوابد، نه اینکه CPU را اشغال کند) و Bounding (محدود کردن حداکثر ظرفیت برای جلوگیری از رشد بی‌رویه حافظه).

---

## بخش ۲: معماری داخلی - چطور کار می‌کند

`BlockingCollection<T>` در فضای نام `System.Collections.Concurrent` قرار دارد و از نظر معماری، الگوی **Decorator Pattern** را پیاده‌سازی می‌کند: یک کالکشن Thread-safe موجود (که `IProducerConsumerCollection<T>` را پیاده کرده) می‌گیرد و رفتار Blocking/Bounding را روی آن اضافه می‌کند، بدون اینکه منطق داخلی آن کالکشن تغییر کند.

پیش‌فرض این کالکشن داخلی `ConcurrentQueue<T>` است (رفتار FIFO)، اما به‌جای آن می‌توان `ConcurrentStack<T>` (رفتار LIFO) یا `ConcurrentBag<T>` (بدون ترتیب مشخص) نیز تزریق کرد:

```csharp
using System.Collections.Concurrent;

// پیش‌فرض: بر پایه ConcurrentQueue، بدون محدودیت ظرفیت
var defaultCollection = new BlockingCollection<string>();

// بر پایه ConcurrentStack با ظرفیت محدود به 1000
var lifoCollection = new BlockingCollection<string>(new ConcurrentStack<string>(), boundedCapacity: 1000);

// بر پایه ConcurrentBag
var bagCollection = new BlockingCollection<string>(new ConcurrentBag<string>());
```

از نظر داخلی، مکانیزم Blocking با ترکیب `SemaphoreSlim` پیاده‌سازی شده است، نه با Polling. یعنی وقتی Thread روی متد `Take()` منتظر می‌ماند، واقعاً در حالت Wait قرار می‌گیرد و توسط Kernel/Scheduler بیدار می‌شود؛ برای آن Thread، تا زمانی که داده جدیدی برسد، هیچ چرخه CPU مصرف نمی‌شود.

---

## بخش ۳: عملیات پایه - Add و Take

دو متد اصلی، `Add` برای تولیدکننده و `Take` برای مصرف‌کننده هستند.

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;

var dataItems = new BlockingCollection<int>(boundedCapacity: 100);

// Producer
var producerTask = Task.Run(() =>
{
    for (var i = 0; i < 500; i++)
    {
        // اگر ظرفیت پر باشد (Count == 100)، اینجا Thread بلاک می‌شود
        dataItems.Add(i);
    }

    // اعلام می‌کند دیگر آیتمی اضافه نمی‌شود
    dataItems.CompleteAdding();
});

// Consumer
var consumerTask = Task.Run(() =>
{
    while (!dataItems.IsCompleted)
    {
        int item;
        try
        {
            // اگر کالکشن خالی باشد، اینجا Thread بلاک می‌شود
            item = dataItems.Take();
        }
        catch (InvalidOperationException)
        {
            // زمانی رخ می‌دهد که بین بررسی IsCompleted و فراخوانی Take
            // یک Thread دیگر CompleteAdding را فراخوانی کرده باشد
            break;
        }

        Console.WriteLine($"Processed: {item}");
    }
});

await Task.WhenAll(producerTask, consumerTask);
```

نکته کلیدی اینجا `CompleteAdding()` است. این متد به کالکشن اعلام می‌کند که دیگر آیتم جدیدی اضافه نخواهد شد، ولی آیتم‌های موجود را پاک نمی‌کند؛ Consumer همچنان می‌تواند باقیمانده را بخواند. `IsCompleted` وقتی `true` می‌شود که هم `CompleteAdding` فراخوانی شده باشد و هم کالکشن خالی شده باشد. این دقیقاً همان الگویی است که در مستندات رسمی مایکروسافت نیز توصیه شده است.

---

## بخش ۴: عملیات غیربلاکه - TryAdd و TryTake

گاهی لازم نیست Thread تا ابد منتظر بماند. برای این موارد از `TryAdd`/`TryTake` همراه با Timeout استفاده می‌شود:

```csharp
var bc = new BlockingCollection<string>(boundedCapacity: 50);

// تلاش برای افزودن به مدت حداکثر 2 ثانیه
var wasAdded = bc.TryAdd("new-item", TimeSpan.FromSeconds(2));
if (!wasAdded)
{
    Console.WriteLine("کالکشن پر است، آیتم رد شد یا باید کار دیگری انجام شود.");
}

// تلاش برای برداشتن به مدت حداکثر 500 میلی‌ثانیه
if (bc.TryTake(out var result, TimeSpan.FromMilliseconds(500)))
{
    Console.WriteLine($"دریافت شد: {result}");
}
else
{
    Console.WriteLine("در بازه زمانی مشخص آیتمی نیامد.");
}
```

این الگو زمانی مفید است که Thread باید بتواند کار دیگری هم انجام دهد، نه اینکه بی‌نهایت منتظر بماند (مثلاً بررسی یک شرط توقف کلی برنامه).

---

## بخش ۵: پشتیبانی از CancellationToken

برای توقف تمیز حلقه‌های Producer/Consumer، به‌جای پرچم‌های دستی `bool`، باید از `CancellationToken` استفاده شود:

```csharp
using var cts = new CancellationTokenSource();
var bc = new BlockingCollection<int>();

var consumer = Task.Run(() =>
{
    try
    {
        foreach (var item in bc.GetConsumingEnumerable(cts.Token))
        {
            Console.WriteLine(item);
        }
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("عملیات مصرف لغو شد.");
    }
});

// جایی دیگر از برنامه
cts.CancelAfter(TimeSpan.FromSeconds(10));
```

هم `Take`/`Add` و هم `TryTake`/`TryAdd` اورلودی دارند که `CancellationToken` می‌گیرند و در صورت لغو، `OperationCanceledException` پرتاب می‌کنند. مدیریت این Exception بر عهده توسعه‌دهنده است؛ کالکشن خودش وضعیت را تمیز نمی‌کند.

---

## بخش ۶: GetConsumingEnumerable - الگوی رایج مصرف

به‌جای نوشتن حلقه دستی با `Take` و مدیریت `IsCompleted`، معمولاً از `GetConsumingEnumerable()` همراه با `foreach` استفاده می‌شود که خودش منطق پایان را مدیریت می‌کند:

```csharp
var bc = new BlockingCollection<int>();

var consumer = Task.Run(() =>
{
    // این حلقه تا زمانی که IsCompleted نشده و آیتمی موجود باشد ادامه دارد
    // در نبود آیتم، بدون مصرف CPU منتظر می‌ماند
    foreach (var item in bc.GetConsumingEnumerable())
    {
        Console.WriteLine($"Processing: {item}");
    }

    Console.WriteLine("تمام آیتم‌ها پردازش شدند.");
});
```

نکته مهم: هر آیتمی که از `GetConsumingEnumerable` عبور کند، از کالکشن حذف می‌شود (Consuming Enumeration). این با `foreach` معمولی روی یک کالکشن (که فقط Read-only Enumeration است) تفاوت دارد.

---

## بخش ۷: کار با چند BlockingCollection همزمان

برای سناریوهای Pipeline (چند مرحله پردازش پشت سر هم)، می‌توان آرایه‌ای از `BlockingCollection<T>` ساخت و از متدهای استاتیک `AddToAny` و `TakeFromAny` استفاده کرد. این متدها به‌محض اینکه یکی از کالکشن‌ها آماده انجام عملیات باشد، از همان استفاده می‌کنند:

```csharp
var collections = new[]
{
    new BlockingCollection<int>(),
    new BlockingCollection<int>()
};

// آیتم را به هر کدام که ابتدا فضای خالی داشته باشد اضافه می‌کند
BlockingCollection<int>.AddToAny(collections, 42);

// از هر کدام که ابتدا آیتم داشته باشد برمی‌دارد
var index = BlockingCollection<int>.TakeFromAny(collections, out var value);
```

این قابلیت کمتر شناخته‌شده است، اما در پیاده‌سازی سیستم‌های Load Balancing بین چند صف کاربرد دارد.

---

## بخش ۸: بایدها و نبایدها

جدول زیر خلاصه نکاتی است که باید در پروژه واقعی رعایت شوند:

| موضوع | باید | نباید |
|---|---|---|
| Dispose | چون `BlockingCollection<T>` از `IDisposable` ارث‌بری می‌کند (به‌خاطر `SemaphoreSlim` داخلی)، حتماً باید با `using` یا `Dispose()` دستی آزاد شود | فراموش نشود که رها نکردنش نشتی منابع سیستم‌عامل (Handle) ایجاد می‌کند |
| پایان تولید | همیشه در انتهای کار Producer، باید `CompleteAdding()` فراخوانی شود | بدون `CompleteAdding`، Consumerهایی که با `GetConsumingEnumerable` کار می‌کنند تا ابد بلاک می‌مانند |
| ظرفیت | برای جلوگیری از مصرف بی‌رویه حافظه، همیشه باید `boundedCapacity` مشخص شود | کالکشن نامحدود در سیستمی که Producer سریع‌تر از Consumer است، منجر به OutOfMemoryException می‌شود |
| Property شمارش | اگر فقط برای گزارش‌گیری تقریبی نیاز باشد، می‌توان از `Count` استفاده کرد | برای تصمیم‌گیری منطقی (مثل «اگر خالی بود فلان کار انجام شود») نباید روی `Count` حساب باز کرد، چون بین خواندن مقدار و اقدام بعدی، مقدار می‌تواند توسط Thread دیگر تغییر کند (Race Condition) |
| مدل همزمانی | برای برنامه‌های مبتنی بر Thread واقعی (مثل Worker Serviceهای کلاسیک، Console App) مناسب است | برای کد Async-heavy مبتنی بر `async/await` که نباید Thread واقعی بلاک شود، باید به‌جایش از `System.Threading.Channels` استفاده شود؛ چون `Take()` این کلاس Thread واقعی OS را می‌خواباند، نه فقط یک Task را |
| مدیریت خطا | همیشه باید `InvalidOperationException` حول فراخوانی `Take`/`Add` بعد از `CompleteAdding` احتمالی مدیریت شود | نباید فرض شود ترتیب بررسی `IsCompleted` و فراخوانی `Take` اتمیک است؛ در بازه بین این دو، وضعیت می‌تواند تغییر کند |
| انتخاب کالکشن داخلی | برای FIFO باید از حالت پیش‌فرض (`ConcurrentQueue`) استفاده شود؛ برای LIFO باید صراحتاً `ConcurrentStack` تزریق شود | نباید فرض شود رفتار پیش‌فرض همیشه ترتیب ورود را حفظ می‌کند، مخصوصاً اگر کالکشن داخلی بعداً تغییر کرده باشد |

---

## بخش ۹: مقایسه با گزینه‌های جایگزین

| ویژگی | BlockingCollection\<T> | Channel\<T> (System.Threading.Channels) | ConcurrentQueue\<T> |
|---|---|---|---|
| مدل همزمانی | Thread-based (بلاک‌کننده Thread واقعی) | Task-based Async (بدون بلاک Thread) | بدون بلاک، نیاز به Polling دستی |
| معرفی‌شده در | .NET Framework 4.0 | .NET Core 3.0 | .NET Framework 4.0 |
| Bounded Capacity | دارد | دارد (BoundedChannelOptions) | ندارد |
| مناسب برای | برنامه‌های کلاسیک Thread/Task.Run، Console/Worker Service | برنامه‌های Async مدرن، Web API، gRPC streaming | مواردی که منطق انتظار به‌صورت async مدیریت می‌شود |
| هزینه بلاک شدن | اشغال یک Thread واقعی از Thread Pool در حالت انتظار | آزاد شدن Thread هنگام انتظار (`await`) | بدون بلاک، ولی نیاز به الگوریتم Backoff دستی |

اگر پروژه روی .NET Core 6/8 قرار دارد و قصد استفاده کامل از async/await وجود دارد، معماری Channel معمولاً از نظر Scalability انتخاب بهتری است، چون Thread Pool را برای انتظار اشغال نمی‌کند.

---

## بخش ۱: Channel چیست و چرا به وجود آمد

`System.Threading.Channels` در .NET Core 3.0 معرفی شد تا مشکل اصلی `BlockingCollection` در دنیای async/await را حل کند: نیاز به اشغال یک Thread واقعی فقط برای «منتظر ماندن». در معماری‌های مدرن مثل ASP.NET Core، Thread Pool منبعی محدود و ارزشمند است؛ اگر هزاران Request هم‌زمان هرکدام یک Thread را برای انتظار روی صف اشغال کنند، برنامه خیلی زودتر از حد انتظار به Thread Starvation می‌رسد.

Channel یک ساختار داده Producer-Consumer است که کاملاً بر پایه `Task` و `async/await` طراحی شده است. وقتی یک Consumer منتظر داده است، به‌جای خواباندن Thread، فقط یک `Task` ناتمام برمی‌گرداند و Thread زیرینش آزاد می‌شود تا کارهای دیگر را انجام دهد. به‌محض رسیدن داده، ادامه کار (Continuation) روی Thread Pool زمان‌بندی می‌شود.

از نظر مفهومی، یک Channel از دو بخش تشکیل شده است: `ChannelWriter<T>` برای نوشتن (سمت Producer) و `ChannelReader<T>` برای خواندن (سمت Consumer). این دو از هم مجزا هستند تا بتوان طبق اصل Interface Segregation، فقط دسترسی لازم را به هر کلاس داد (مثلاً یک متد فقط `ChannelReader<T>` بگیرد و امکان نوشتن نداشته باشد).

---

## بخش ۲: ساخت Channel - Bounded در برابر Unbounded

Channel از طریق کلاس استاتیک `Channel` ساخته می‌شود، نه با `new` مستقیم، چون خودِ `Channel<T>` انتزاعی (Abstract) است. این استفاده از **Factory Pattern** است.

```csharp
using System.Threading.Channels;

// کانال بدون محدودیت ظرفیت - باید مراقب رشد بی‌رویه حافظه بود
var unboundedChannel = Channel.CreateUnbounded<int>();

// کانال با ظرفیت محدود به 100 آیتم
var boundedChannel = Channel.CreateBounded<int>(capacity: 100);

// کانال محدود با تنظیمات دقیق‌تر
var configuredChannel = Channel.CreateBounded<int>(new BoundedChannelOptions(capacity: 100)
{
    // رفتار وقتی کانال پر است
    FullMode = BoundedChannelFullMode.Wait,

    // اجازه فقط یک Writer در آن واحد (بهینه‌سازی داخلی)
    SingleWriter = false,

    // اجازه فقط یک Reader در آن واحد (بهینه‌سازی داخلی)
    SingleReader = false
});
```

نکته مهم عملکردی: اگر مطمئن باشیم که فقط یک Producer یا فقط یک Consumer وجود دارد، باید حتماً `SingleWriter`/`SingleReader` روی `true` قرار گیرد. این کار به Channel اجازه می‌دهد از الگوریتم‌های داخلی سبک‌تر (بدون نیاز به Synchronization اضافی) استفاده کند و Throughput را افزایش دهد.

---

## بخش ۳: رفتار BoundedChannelFullMode - چه اتفاقی می‌افتد وقتی کانال پر است

وقتی از Bounded Channel استفاده می‌شود، باید مشخص شود که هنگام پر شدن ظرفیت، چه رفتاری مورد انتظار است. این دقیقاً همان بحث Backpressure است که در `BlockingCollection` نیز وجود داشت، اما اینجا چهار حالت وجود دارد، نه فقط یک حالت بلاک‌کننده:

| مقدار FullMode | رفتار | کاربرد مناسب |
|---|---|---|
| Wait (پیش‌فرض) | فراخوانی `WriteAsync` منتظر می‌ماند تا جا باز شود؛ `TryWrite` بلافاصله `false` برمی‌گرداند | زمانی که هیچ داده‌ای نباید گم شود (مثلاً صف تراکنش‌های مالی) |
| DropOldest | قدیمی‌ترین آیتم موجود در کانال حذف می‌شود تا جای آیتم جدید باز شود | داده‌های Real-time که فقط آخرین مقدار اهمیت دارد (مثل تله‌متری سنسور) |
| DropNewest | جدیدترین آیتم موجود در کانال (نه آیتم در حال نوشتن) حذف می‌شود | کاربرد کمتری دارد؛ برای حفظ داده‌های قدیمی‌تر |
| DropWrite | خودِ آیتمی که در حال نوشتن است، رها می‌شود و کانال بدون تغییر می‌ماند | زمانی که از دست رفتن پیام‌های جدید تازه‌وارد قابل قبول است |

```csharp
var telemetryChannel = Channel.CreateBounded<SensorReading>(new BoundedChannelOptions(50)
{
    FullMode = BoundedChannelFullMode.DropOldest
});
```

---

## بخش ۴: نوشتن در Channel - سمت Producer

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

public sealed class OrderProducer
{
    private readonly ChannelWriter<Order> _writer;

    public OrderProducer(ChannelWriter<Order> writer)
    {
        _writer = writer;
    }

    public async Task ProduceAsync(int count, CancellationToken cancellationToken)
    {
        for (var i = 0; i < count; i++)
        {
            var order = new Order(i);

            // اگر کانال پر باشد (در حالت Wait)، اینجا Thread بلاک نمی‌شود
            // بلکه Task تا زمان باز شدن جا، ناتمام باقی می‌ماند
            await _writer.WriteAsync(order, cancellationToken);
        }

        // اعلام پایان نوشتن؛ معادل CompleteAdding در BlockingCollection
        _writer.Complete();
    }
}

public sealed record Order(int Id);
```

اگر بدون انتظار نوشتن مدنظر باشد و فقط بررسی موفقیت آن اهمیت داشته باشد، باید از `TryWrite` استفاده شود:

```csharp
if (!writer.TryWrite(order))
{
    // کانال پر است (در حالت Wait) یا بسته شده است
}
```

نکته‌ای که باید در Exception Handling رعایت شود: اگر بعد از `Complete()` دوباره تلاش برای نوشتن انجام شود، `ChannelClosedException` پرتاب می‌شود. اگر حین نوشتن خطایی رخ دهد که باید به Consumer اطلاع‌رسانی شود، می‌توان آن را به `Complete(Exception)` پاس داد:

```csharp
try
{
    await ProduceDataAsync(writer);
    writer.Complete();
}
catch (Exception ex)
{
    // این Exception هنگام خواندن توسط Consumer دوباره پرتاب می‌شود
    writer.Complete(ex);
}
```

---

## بخش ۵: خواندن از Channel - سمت Consumer

سه الگوی رایج برای خواندن وجود دارد. مهم است بدانیم هرکدام کجا مناسب‌ترند.

**الگوی اول - IAsyncEnumerable (توصیه‌شده در .NET Core 3.0+):**

```csharp
public sealed class OrderConsumer
{
    private readonly ChannelReader<Order> _reader;

    public OrderConsumer(ChannelReader<Order> reader)
    {
        _reader = reader;
    }

    public async Task ConsumeAsync(CancellationToken cancellationToken)
    {
        // ReadAllAsync خودش منتظر داده جدید می‌ماند و وقتی کانال Complete و خالی شد، خارج می‌شود
        await foreach (var order in _reader.ReadAllAsync(cancellationToken))
        {
            await ProcessOrderAsync(order);
        }
    }

    private Task ProcessOrderAsync(Order order)
    {
        Console.WriteLine($"Processing order {order.Id}");
        return Task.CompletedTask;
    }
}
```

**الگوی دوم - WaitToReadAsync + TryRead (وقتی نیاز به کنترل دقیق‌تر باشد):**

```csharp
while (await reader.WaitToReadAsync(cancellationToken))
{
    while (reader.TryRead(out var order))
    {
        await ProcessOrderAsync(order);
    }
}
```

`WaitToReadAsync` وقتی داده‌ای موجود باشد `true` و وقتی کانال بسته و خالی شده باشد `false` برمی‌گرداند. حلقه داخلی با `TryRead` لازم است، چون در سناریوی چند Consumer، ممکن است بین اطلاع «داده موجود است» و لحظه واقعی خواندن، Consumer دیگری آن را برداشته باشد.

**الگوی سوم - ReadAsync مستقیم (وقتی دقیقاً یک آیتم مدنظر است):**

```csharp
try
{
    var order = await reader.ReadAsync(cancellationToken);
    await ProcessOrderAsync(order);
}
catch (ChannelClosedException)
{
    Console.WriteLine("کانال بسته شده و دیگر داده‌ای نیست.");
}
```

---

## بخش ۶: پیاده‌سازی کامل الگوی Producer-Consumer

این نمونه، یک Pipeline کامل با یک Producer و چند Consumer موازی را نشان می‌دهد؛ الگویی که در پردازش صف پیام یا لاگ در Worker Service پرکاربرد است:

```csharp
using System;
using System.Threading;
using System.Threading.Channels;
using System.Threading.Tasks;

public sealed class LogProcessingPipeline
{
    private readonly Channel<LogEntry> _channel;

    public LogProcessingPipeline(int boundedCapacity)
    {
        _channel = Channel.CreateBounded<LogEntry>(new BoundedChannelOptions(boundedCapacity)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true,
            SingleReader = false
        });
    }

    public async Task RunAsync(int consumerCount, CancellationToken cancellationToken)
    {
        var producerTask = ProduceAsync(cancellationToken);

        var consumerTasks = new Task[consumerCount];
        for (var i = 0; i < consumerCount; i++)
        {
            var consumerId = i;
            consumerTasks[i] = ConsumeAsync(consumerId, cancellationToken);
        }

        await producerTask;
        await Task.WhenAll(consumerTasks);
    }

    private async Task ProduceAsync(CancellationToken cancellationToken)
    {
        var writer = _channel.Writer;

        try
        {
            for (var i = 0; i < 1000; i++)
            {
                await writer.WriteAsync(new LogEntry(i, $"Log message {i}"), cancellationToken);
            }
        }
        finally
        {
            writer.Complete();
        }
    }

    private async Task ConsumeAsync(int consumerId, CancellationToken cancellationToken)
    {
        var reader = _channel.Reader;

        await foreach (var entry in reader.ReadAllAsync(cancellationToken))
        {
            Console.WriteLine($"[Consumer {consumerId}] {entry.Message}");
        }
    }
}

public sealed record LogEntry(int Id, string Message);
```

توجه شود که `writer.Complete()` داخل بلوک `finally` قرار گرفته است؛ این کار تضمین می‌کند که حتی اگر در حلقه Producer استثنایی رخ دهد، Consumerها تا ابد در انتظار نمی‌مانند.

---

## بخش ۷: بایدها و نبایدها

| موضوع | باید | نباید |
|---|---|---|
| بستن کانال | همیشه بعد از پایان تولید، باید `writer.Complete()` فراخوانی شود (ترجیحاً در `finally`) | فراموش نشود؛ بدون آن، `ReadAllAsync` و `WaitToReadAsync` تا ابد منتظر می‌مانند |
| انتخاب Bounded/Unbounded | برای جلوگیری از فشار حافظه در تولید سریع‌تر از مصرف، باید از `CreateBounded` با ظرفیت مشخص استفاده شود | از `CreateUnbounded` در سیستم Production بدون بررسی نرخ تولید/مصرف نباید استفاده شود |
| SingleReader/SingleWriter | اگر فقط یک Producer یا Consumer وجود دارد، باید این پرچم‌ها `true` باشند تا بهینه‌سازی داخلی فعال شود | این پرچم‌ها نباید اشتباه `true` قرار گیرند وقتی چند Thread واقعاً می‌نویسند/می‌خوانند؛ این باعث Corruption داده می‌شود |
| فراخوانی Sync | همیشه باید از `await` روی `WriteAsync`/`ReadAsync` استفاده شود | هرگز نباید از `.AsTask().Result` یا `.Wait()` استفاده شود؛ دقیقاً همان ریسک Deadlock معروف async-over-sync در ASP.NET را وارد می‌کند |
| مدیریت خطا | خطای سمت Producer باید با `writer.Complete(exception)` به Consumer منتقل شود | خطا نباید در Producer قورت داده شود؛ Consumer باید بداند پردازش به‌طور غیرعادی متوقف شده است |
| انتخاب FullMode | برای داده‌های حیاتی باید از `Wait` (پیش‌فرض) و برای داده‌های Real-time که فقط آخرین مقدار اهمیت دارد باید از `DropOldest` استفاده شود | نباید فرض شود `DropOldest`/`DropNewest` همیشه بی‌خطرند؛ اگر گم شدن داده در کسب‌وکار غیرقابل قبول است، فقط باید `Wait` انتخاب شود |
| انتخاب بین Channel و BlockingCollection | برای معماری Async-first (ASP.NET Core، gRPC streaming، SignalR) باید از Channel استفاده شود | در کد کاملاً Thread-based قدیمی (بدون async/await)، نباید به‌زور Channel جایگزین `BlockingCollection` شود؛ سربار Task Scheduling بدون فایده اضافه می‌شود |

---

## بخش ۸: کاربرد واقعی - SignalR Streaming

یکی از کاربردهای رسمی و شناخته‌شده Channel، پیاده‌سازی Streaming در SignalR است، جایی که سرور می‌تواند داده را به‌صورت پیوسته به کلاینت ارسال کند:

```csharp
public async IAsyncEnumerable<SensorReading> StreamReadings(
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    var channel = Channel.CreateBounded<SensorReading>(10);

    _ = Task.Run(async () =>
    {
        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                var reading = await ReadFromSensorAsync();
                await channel.Writer.WriteAsync(reading, cancellationToken);
            }
        }
        finally
        {
            channel.Writer.Complete();
        }
    }, cancellationToken);

    await foreach (var reading in channel.Reader.ReadAllAsync(cancellationToken))
    {
        yield return reading;
    }
}
```

این الگو دقیقاً همان چیزی است که در مستندات و نمونه‌های رسمی SignalR برای Streaming کلاینت-به-سرور و سرور-به-کلاینت استفاده شده است.
