---
title: "چه زمانی از BlockingCollection و چه زمانی از ConcurrentQueue استفاده کنیم؟"
categories:
  - Net
tags:
  - net
  - collection
  - queue
---

## بخش ۱: مسئله چیست و BlockingCollection چرا به وجود آمد

فرض کن یک برنامه داری که در آن چند Thread تولیدکننده داده (Producer) دارند و چند Thread مصرف‌کننده (Consumer) که باید آن داده‌ها را پردازش کنند. مثال کلاسیک: یک Thread فایل‌های لاگ را می‌خواند و در صف می‌گذارد، Thread دیگری آن‌ها را پردازش و در دیتابیس ذخیره می‌کند.

اگر بخواهی این کار را با یک `Queue<T>` معمولی پیاده کنی، دو مشکل اساسی پیدا می‌کنی:

اول، `Queue<T>` ذاتاً Thread-safe نیست. اگر دو Thread همزمان به آن `Enqueue`/`Dequeue` بزنند، ساختار داخلی صف (که بر پایه آرایه است) خراب می‌شود. راه‌حل اولیه، قفل‌گذاری دستی با `lock` است، اما این کار خودش دو ریسک دارد: افت شدید Performance به خاطر Contention روی قفل، و احتمال Deadlock در صورت مدیریت نادرست.

دوم، حتی اگر مشکل Thread-safety را با `ConcurrentQueue<T>` حل کنی، مشکل دیگری باقی می‌ماند: وقتی صف خالی است، Consumer باید چه کار کند؟ اگر پیوسته در یک حلقه `while` چک کند که آیا آیتم جدید آمده یا نه (Polling/Busy-Waiting)، منابع CPU را بی‌دلیل می‌سوزاند. اگر هم بخواهی خودت با `ManualResetEvent` یا `Monitor.Wait/Pulse` منطق "صبر کن تا داده بیاید" را پیاده‌سازی کنی، کد پیچیده و مستعد باگ‌های Race Condition می‌شود.

`BlockingCollection<T>` دقیقاً برای حل همین دو مسئله در .NET Framework 4.0 معرفی شد: پیاده‌سازی آماده و تست‌شده الگوی Producer-Consumer، به‌همراه قابلیت Blocking خودکار (Thread واقعاً می‌خوابد، نه اینکه CPU را اشغال کند) و Bounding (محدود کردن حداکثر ظرفیت برای جلوگیری از رشد بی‌رویه حافظه). 

***

## بخش ۲: معماری داخلی — چطور کار می‌کند

`BlockingCollection<T>` در فضای نام `System.Collections.Concurrent` قرار دارد و از نظر معماری، الگوی **Decorator Pattern** را پیاده‌سازی می‌کند: یک کالکشن Thread-safe موجود (که `IProducerConsumerCollection<T>` را پیاده کرده) می‌گیرد و رفتار Blocking/Bounding را روی آن اضافه می‌کند، بدون اینکه منطق داخلی آن کالکشن را تغییر دهد.

پیش‌فرض این کالکشن داخلی `ConcurrentQueue<T>` است (رفتار FIFO)، اما می‌توانی به‌جای آن `ConcurrentStack<T>` (رفتار LIFO) یا `ConcurrentBag<T>` (بدون ترتیب مشخص) هم تزریق کنی:

```csharp
using System.Collections.Concurrent;

// پیش‌فرض: بر پایه ConcurrentQueue، بدون محدودیت ظرفیت
var defaultCollection = new BlockingCollection<string>();

// بر پایه ConcurrentStack با ظرفیت محدود به 1000
var lifoCollection = new BlockingCollection<string>(new ConcurrentStack<string>(), boundedCapacity: 1000);

// بر پایه ConcurrentBag
var bagCollection = new BlockingCollection<string>(new ConcurrentBag<string>());
```

از نظر داخلی، مکانیزم Blocking با ترکیب `SemaphoreSlim` پیاده‌سازی شده است، نه با Polling. یعنی وقتی Thread روی متد `Take()` منتظر می‌ماند، واقعاً در حالت Wait قرار می‌گیرد و توسط Kernel/Scheduler بیدار می‌شود؛ CPU برای آن Thread چرخه‌ای مصرف نمی‌کند تا زمانی که داده جدیدی برسد. 

***

## بخش ۳: عملیات پایه — Add و Take

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
            // زمانی رخ می‌دهد که بین چک IsCompleted و فراخوانی Take
            // یک Thread دیگر CompleteAdding را صدا زده باشد
            break;
        }

        Console.WriteLine($"Processed: {item}");
    }
});

await Task.WhenAll(producerTask, consumerTask);
```

نکته کلیدی اینجا `CompleteAdding()` است. این متد به کالکشن می‌گوید "دیگر آیتم جدیدی اضافه نخواهد شد"، ولی آیتم‌های موجود را پاک نمی‌کند؛ Consumer همچنان می‌تواند باقیمانده را بخواند. `IsCompleted` وقتی `true` می‌شود که هم `CompleteAdding` صدا زده شده باشد و هم کالکشن خالی شده باشد. این دقیقاً همان الگویی است که در مستندات رسمی مایکروسافت هم توصیه شده است. 

***

## بخش ۴: عملیات غیربلاکه — TryAdd و TryTake

گاهی نمی‌خواهی Thread تا ابد منتظر بماند. برای این موارد از `TryAdd`/`TryTake` با Timeout استفاده می‌شود:

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

این الگو زمانی مفید است که Thread باید بتواند کار دیگری هم انجام دهد، نه اینکه بی‌نهایت منتظر بماند (مثلاً چک کردن یک شرط توقف کلی برنامه). 

***

## بخش ۵: پشتیبانی از CancellationToken

برای توقف تمیز حلقه‌های Producer/Consumer، به‌جای پرچم‌های دستی `bool`، از `CancellationToken` استفاده کن:

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

هم `Take`/`Add` و هم `TryTake`/`TryAdd` اورلودی دارند که `CancellationToken` می‌گیرند و در صورت لغو، `OperationCanceledException` پرتاب می‌کنند. مدیریت این Exception به عهده توسعه‌دهنده است؛ کالکشن خودش وضعیت را تمیز نمی‌کند. 

***

## بخش ۶: GetConsumingEnumerable — الگوی رایج مصرف

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

نکته مهم: هر آیتمی که از `GetConsumingEnumerable` عبور کند، از کالکشن حذف می‌شود (Consuming Enumeration). این با `foreach` معمولی روی یک کالکشن (که فقط Read-only Enumeration است) فرق دارد. 

***

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

***

## بخش ۸: بایدها و نبایدها

جدول زیر خلاصه نکاتی است که در پروژه واقعی باید رعایت شوند:

| موضوع | باید | نباید |
|---|---|---|
| Dispose | چون `BlockingCollection<T>` از `IDisposable` ارث‌بری می‌کند (به‌خاطر `SemaphoreSlim` داخلی)، حتماً با `using` یا `Dispose()` دستی آزادش کن | فراموش نکن که رها کردنش نشتی منابع سیستم‌عامل (Handle) ایجاد می‌کند |
| پایان تولید | همیشه در انتهای کار Producer، `CompleteAdding()` را صدا بزن | بدون `CompleteAdding`، Consumerهایی که با `GetConsumingEnumerable` کار می‌کنند تا ابد بلاک می‌مانند |
| ظرفیت | برای جلوگیری از مصرف بی‌رویه حافظه، همیشه `boundedCapacity` مشخص کن | کالکشن نامحدود در سیستمی که Producer سریع‌تر از Consumer است، منجر به OutOfMemoryException می‌شود |
| Property شمارش | اگر فقط برای گزارش‌گیری تقریبی نیاز داری از `Count` استفاده کن | برای تصمیم‌گیری منطقی (مثل "اگر خالی بود فلان کار را بکن") روی `Count` حساب نکن، چون بین خواندن مقدار و اقدام بعدی، مقدار می‌تواند توسط Thread دیگر تغییر کند (Race Condition) |
| مدل همزمانی | برای برنامه‌های مبتنی بر Thread واقعی (مثل Worker Serviceهای کلاسیک، Console App) مناسب است | برای کد Async-heavy مبتنی بر `async/await` که نمی‌خواهی Thread واقعی را بلاک کنی، به‌جایش از `System.Threading.Channels` استفاده کن؛ چون `Take()` این کلاس Thread واقعی OS را می‌خواباند، نه فقط یک Task را |
| مدیریت خطا | همیشه `InvalidOperationException` را حول فراخوانی `Take`/`Add` بعد از `CompleteAdding` احتمالی مدیریت کن | فرض نکن ترتیب چک `IsCompleted` و فراخوانی `Take` اتمیک است؛ در بازه بین این دو، وضعیت می‌تواند تغییر کند |
| انتخاب کالکشن داخلی | برای FIFO از حالت پیش‌فرض (`ConcurrentQueue`) استفاده کن؛ برای LIFO صراحتاً `ConcurrentStack` تزریق کن | فرض نکن رفتار پیش‌فرض همیشه ترتیب ورود را حفظ می‌کند، مخصوصاً اگر کسی بعداً کالکشن داخلی را عوض کرده باشد |

***

## بخش ۹: مقایسه با گزینه‌های جایگزین

این بخش را حتماً در انتهای مقاله بگذار تا خواننده گمراه نشود که `BlockingCollection` تنها راه‌حل است:

| ویژگی | BlockingCollection\<T> | Channel\<T> (System.Threading.Channels) | ConcurrentQueue\<T> |
|---|---|---|---|
| مدل همزمانی | Thread-based (بلاک‌کننده Thread واقعی) | Task-based Async (بدون بلاک Thread) | بدون بلاک، نیاز به Polling دستی |
| معرفی‌شده در | .NET Framework 4.0 | .NET Core 3.0 | .NET Framework 4.0 |
| Bounded Capacity | دارد | دارد (BoundedChannelOptions) | ندارد |
| مناسب برای | برنامه‌های کلاسیک Thread/Task.Run، Console/Worker Service | برنامه‌های Async مدرن، Web API، gRPC streaming | مواردی که خودت منطق انتظار را با async مدیریت می‌کنی |
| هزینه بلاک شدن | اشغال یک Thread واقعی از Thread Pool در حالت انتظار | آزاد کردن Thread هنگام انتظار (`await`) | بدون بلاک، ولی نیاز به الگوریتم Backoff دستی |

اگر پروژه‌ات روی .NET Core 6/8 است و می‌خواهی از async/await به‌صورت کامل بهره ببری، معماری Channel معمولاً انتخاب بهتری از نظر Scalability است، چون Thread Pool را برای انتظار اشغال نمی‌کند.

---

## بخش ۱: Channel چیست و چرا به وجود آمد

`System.Threading.Channels` در .NET Core 3.0 معرفی شد تا مشکل اصلی `BlockingCollection` در دنیای async/await را حل کند: نیاز به اشغال یک Thread واقعی فقط برای "منتظر ماندن". در معماری‌های مدرن مثل ASP.NET Core، Thread Pool منبعی محدود و ارزشمند است؛ اگر هزاران Request هم‌زمان هرکدام یک Thread را برای انتظار روی صف اشغال کنند، برنامه خیلی زودتر از حد انتظار به Thread Starvation می‌رسد.

Channel یک ساختار داده Producer-Consumer است که کاملاً بر پایه `Task` و `async/await` طراحی شده. وقتی یک Consumer منتظر داده است، به‌جای خواباندن Thread، فقط یک `Task` ناتمام برمی‌گرداند و Thread زیرینش آزاد می‌شود تا کارهای دیگر را انجام دهد. به محض رسیدن داده، ادامه کار (Continuation) روی Thread Pool زمان‌بندی می‌شود. 

از نظر مفهومی، یک Channel از دو بخش تشکیل شده: `ChannelWriter<T>` برای نوشتن (سمت Producer) و `ChannelReader<T>` برای خواندن (سمت Consumer). این دو، از هم مجزا هستند تا بتوانی طبق اصل Interface Segregation، فقط دسترسی لازم را به هر کلاس بدهی (مثلاً یک متد فقط `ChannelReader<T>` می‌گیرد و امکان نوشتن ندارد).

***

## بخش ۲: ساخت Channel — Bounded در برابر Unbounded

Channel از طریق کلاس استاتیک `Channel` ساخته می‌شود، نه با `new` مستقیم، چون خودِ `Channel<T>` انتزاعی (Abstract) است. این استفاده از **Factory Pattern** است.

```csharp
using System.Threading.Channels;

// کانال بدون محدودیت ظرفیت - مراقب رشد بی‌رویه حافظه باش
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

نکته مهم عملکردی: اگر مطمئنی فقط یک Producer یا فقط یک Consumer داری، حتماً `SingleWriter`/`SingleReader` را روی `true` بگذار. این کار به Channel اجازه می‌دهد از الگوریتم‌های داخلی سبک‌تر (بدون نیاز به Synchronization اضافی) استفاده کند و Throughput را افزایش می‌دهد. 

***

## بخش ۳: رفتار BoundedChannelFullMode — چه اتفاقی می‌افتد وقتی کانال پر است

وقتی از Bounded Channel استفاده می‌کنی، باید مشخص کنی وقتی ظرفیت پر شد چه رفتاری می‌خواهی. این دقیقاً همان بحث Backpressure است که در `BlockingCollection` هم وجود داشت، اما اینجا چهار حالت داری، نه فقط یک حالت بلاک‌کننده:

| مقدار FullMode | رفتار | کاربرد مناسب |
|---|---|---|
| Wait (پیش‌فرض) | فراخوانی `WriteAsync` منتظر می‌ماند تا جا باز شود؛ `TryWrite` بلافاصله `false` برمی‌گرداند | زمانی که هیچ داده‌ای نباید گم شود (مثلاً صف تراکنش‌های مالی) |
| DropOldest | قدیمی‌ترین آیتم موجود در کانال حذف می‌شود تا جای آیتم جدید باز شود | داده‌های Real-time که فقط آخرین مقدار مهم است (مثل تله‌متری سنسور) |
| DropNewest | جدیدترین آیتم موجود در کانال (نه آیتم در حال نوشتن) حذف می‌شود | کاربرد کمتری دارد؛ برای حفظ داده‌های قدیمی‌تر |
| DropWrite | خودِ آیتمی که در حال نوشتن است، رها می‌شود و کانال بدون تغییر می‌ماند | زمانی که از دست رفتن پیام‌های جدید تازه‌وارد قابل قبول است |

```csharp
var telemetryChannel = Channel.CreateBounded<SensorReading>(new BoundedChannelOptions(50)
{
    FullMode = BoundedChannelFullMode.DropOldest
});
```

این جدول دقیقاً بر اساس مستندات رسمی مایکروسافت است. 

***

## بخش ۴: نوشتن در Channel — سمت Producer

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

اگر بخواهی بدون انتظار بنویسی و فقط بخواهی بدانی نوشتن موفق بوده یا نه، از `TryWrite` استفاده کن:

```csharp
if (!writer.TryWrite(order))
{
    // کانال پر است (در حالت Wait) یا بسته شده است
}
```

نکته‌ای که باید در Exception Handling رعایت شود: اگر بعد از `Complete()` دوباره تلاش برای نوشتن شود، `ChannelClosedException` پرتاب می‌شود. اگر حین نوشتن خطایی رخ دهد که باید به Consumer اطلاع‌رسانی شود، می‌توانی آن را به `Complete(Exception)` پاس بدهی:

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

***

## بخش ۵: خواندن از Channel — سمت Consumer

سه الگوی رایج برای خواندن وجود دارد. مهم است بدانی هرکدام کجا مناسب‌ترند.

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

**الگوی دوم - WaitToReadAsync + TryRead (وقتی نیاز به کنترل دقیق‌تر داری):**

```csharp
while (await reader.WaitToReadAsync(cancellationToken))
{
    while (reader.TryRead(out var order))
    {
        await ProcessOrderAsync(order);
    }
}
```

`WaitToReadAsync` وقتی داده‌ای موجود باشد `true` و وقتی کانال بسته و خالی شده باشد `false` برمی‌گرداند. حلقه داخلی با `TryRead` لازم است، چون در سناریوی چند Consumer، ممکن است بین اطلاع "داده موجود است" و لحظه واقعی خواندن، Consumer دیگری آن را برداشته باشد. 

**الگوی سوم - ReadAsync مستقیم (وقتی دقیقاً یک آیتم می‌خواهی):**

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

***

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

توجه کن که `writer.Complete()` داخل بلوک `finally` قرار گرفته؛ این کار تضمین می‌کند حتی اگر در حلقه Producer استثنایی رخ دهد، Consumerها تا ابد در انتظار نمی‌مانند.

***

## بخش ۷: بایدها و نبایدها

| موضوع | باید | نباید |
|---|---|---|
| بستن کانال | همیشه بعد از پایان تولید، `writer.Complete()` را صدا بزن (ترجیحاً در `finally`) | فراموش نکن؛ بدون آن، `ReadAllAsync` و `WaitToReadAsync` تا ابد منتظر می‌مانند |
| انتخاب Bounded/Unbounded | برای جلوگیری از فشار حافظه در تولید سریع‌تر از مصرف، از `CreateBounded` با ظرفیت مشخص استفاده کن | از `CreateUnbounded` در سیستم Production بدون بررسی نرخ تولید/مصرف استفاده نکن |
| SingleReader/SingleWriter | اگر فقط یک Producer یا Consumer داری، این پرچم‌ها را `true` کن تا بهینه‌سازی داخلی فعال شود | این پرچم‌ها را اشتباه `true` نگذار وقتی چند Thread واقعاً می‌نویسند/می‌خوانند؛ این باعث Corruption داده می‌شود |
| فراخوانی Sync | همیشه از `await` روی `WriteAsync`/`ReadAsync` استفاده کن | هرگز از `.AsTask().Result` یا `.Wait()` استفاده نکن؛ دقیقاً همان ریسک Deadlock معروف async-over-sync در ASP.NET را وارد می‌کند |
| مدیریت خطا | خطای سمت Producer را با `writer.Complete(exception)` به Consumer منتقل کن | خطا را در Producer قورت نده؛ Consumer باید بداند پردازش به‌طور غیرعادی متوقف شده |
| انتخاب FullMode | برای داده‌های حیاتی از `Wait` (پیش‌فرض) و برای داده‌های Real-time که فقط آخرین مقدار مهم است از `DropOldest` استفاده کن | فرض نکن `DropOldest`/`DropNewest` همیشه بی‌خطرند؛ اگر گم شدن داده در کسب‌وکارت غیرقابل قبول است فقط `Wait` را انتخاب کن |
| انتخاب بین Channel و BlockingCollection | برای معماری Async-first (ASP.NET Core، gRPC streaming، SignalR) از Channel استفاده کن | در کد کاملاً Thread-based قدیمی (بدون async/await)، به‌زور Channel را جایگزین `BlockingCollection` نکن؛ سربار Task Scheduling بدون فایده اضافه می‌شود |

***

## بخش ۸: کاربرد واقعی — SignalR Streaming

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