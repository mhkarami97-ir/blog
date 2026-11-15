---
title: "چرا Rollback در EF Core تغییرات DbContext را برنمی‌گرداند؟"
description: "بررسی باگ رایج در EF Core: بعد از Rollback، تغییرات tracked در DbContext باقی می‌مانند و در SaveChanges بعدی ذخیره می‌شوند."
categories: [dotnet, ef-core]
tags: [ef-core, dotnet, transaction, change-tracker, unit-of-work, outbox]
---

در Code Review به این مورد برخوردم: handler ای که در یک حلقه چند آیتم را پردازش می‌کند، در `catch` تراکنش را Rollback می‌کند و به iteration بعدی می‌رود. ظاهر کد درست است، اما می‌تواند داده را ناسازگار کند. در این مطلب علت، سناریو و راه‌حل‌ها را مرحله‌به‌مرحله بررسی می‌کنیم.

## مسئله چیست؟

Rollback فقط **دیتابیس** را برمی‌گرداند، نه **حافظه** را. EF Core تغییرات entityها را در `ChangeTracker` نگه می‌دارد. این وضعیت تا یکی از این اتفاق‌ها باقی می‌ماند:

- DbContext dispose شود.
- Change Tracker پاک شود (`ChangeTracker.Clear()`).
- Entity صریحاً detach شود.

پس اگر بعد از `RollbackAsync()` از همان context استفاده کنید، entityهای `Modified` و `Added` هنوز آنجا هستند.

## سناریوی خطا

فرض کنید handler ای برای تطبیق (reconciliation) revokeها دارید که برای هر آیتم، وضعیت را تغییر می‌دهد و یک پیام Outbox اضافه می‌کند:

```csharp
foreach (var item in items)
{
    var transaction = await unitOfWork.BeginTransactionAsync(cancellationToken);
    try
    {
        item.Revoke();
        outboxRepository.Add(OutboxMessage.Create(item));
        await externalGateway.NotifyAsync(item, cancellationToken);
        await unitOfWork.SaveChangesAsync(cancellationToken);
        await transaction.CommitAsync(cancellationToken);
    }
    catch (Exception)
    {
        await transaction.RollbackAsync(cancellationToken);
    }
}
```

اتفاقات به ترتیب:

1. در iteration اول، `Revoke()` و افزودن Outbox در Change Tracker ثبت می‌شوند.
2. خطایی رخ می‌دهد (مثلاً timeout سرویس خارجی یا شکست `SaveChanges`).
3. `Rollback` اجرا می‌شود، ولی Change Tracker دست‌نخورده می‌ماند.
4. در iteration دوم، `SaveChangesAsync` روی **همان context** صدا زده می‌شود و تغییرات iteration اول هم ذخیره می‌شوند.

نتیجه: یک revoke یا پیام Outbox برای عملیاتی ثبت می‌شود که در واقع شکست خورده بود. این ناسازگاری بعداً به‌سختی ردیابی می‌شود.

## نمودار جریان

```text
Iteration 1:  Revoke() + Outbox.Add()  -> Tracker: [Modified, Added]
              خطا -> Rollback (فقط DB)  -> Tracker: [Modified, Added]  (هنوز!)
Iteration 2:  تغییر جدید + SaveChanges -> هر سه تغییر نوشته می‌شوند
```

## آیا Rollback صریح لازم است؟

طبق مستندات EF Core، اگر تراکنش Commit نشود، عملیات آن روی دیتابیس اعمال نمی‌شود. بنابراین Dispose یا Rollback تراکنش، دیتابیس را درست نگه می‌دارد. اما هیچ‌کدام وضعیت Change Tracker را ریست نمی‌کنند. پس سؤال اصلی «Rollback بنویسیم یا نه» نیست؛ سؤال این است که **context بعد از شکست چه می‌شود**.

## راه‌حل ۱: Scope مستقل برای هر iteration (پیشنهادی)

مستندات EF Core می‌گویند DbContext برای عمر کوتاه و یک unit-of-work طراحی شده است. پس برای هر آیتم یک scope جدید بسازید تا tracker و تراکنش با هم از بین بروند:

```csharp
public sealed class FastPaymentRevokeReconciliationHandler
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<FastPaymentRevokeReconciliationHandler> _logger;

    public FastPaymentRevokeReconciliationHandler(
        IServiceScopeFactory scopeFactory,
        ILogger<FastPaymentRevokeReconciliationHandler> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public async Task HandleAsync(IReadOnlyList<Guid> ids, CancellationToken cancellationToken)
    {
        foreach (var id in ids)
        {
            await ProcessSingleAsync(id, cancellationToken);
        }
    }

    private async Task ProcessSingleAsync(Guid id, CancellationToken cancellationToken)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var processor = scope.ServiceProvider.GetRequiredService<IRevokeProcessor>();

        try
        {
            await processor.ProcessAsync(id, cancellationToken);
        }
        catch (Exception exception)
        {
            _logger.LogError(exception, "Revoke reconciliation failed for {Id}", id);
        }
    }
}
```

در `IRevokeProcessor` تراکنش را شروع و Commit کنید. اگر خطا رخ دهد، با dispose شدن scope هم تراکنش و هم Change Tracker از بین می‌روند. مزیت این روش: خطای یک آیتم روی آیتم‌های دیگر اثر نمی‌گذارد.

## راه‌حل ۲: ریست کردن داخل UnitOfWork

اگر ناچارید UnitOfWork بین iterationها زنده بماند، Rollback باید هر دو کار را انجام دهد:

```csharp
public async Task RollbackAsync(CancellationToken cancellationToken = default)
{
    try
    {
        if (_transaction is not null)
        {
            await _transaction.RollbackAsync(cancellationToken);
        }
    }
    finally
    {
        _dbContext.ChangeTracker.Clear();
        await DisposeTransactionAsync();
    }
}
```

دو هشدار مهم:

- `Clear()` **همه** entityهای tracked را detach می‌کند، نه فقط entity خراب. هر چیزی را که بیرون از حلقه لود کرده‌اید و بعداً لازم دارید، باید دوباره لود کنید.
- `Clear()` به دیتابیس دست نمی‌زند و فقط حافظه context را خالی می‌کند.

## مقایسه دو راه‌حل

| معیار | Scope جدید | Clear در UnitOfWork |
|---|---|---|
| ایزولاسیون | کامل | وابسته به نظم کد |
| ریسک فراموشی | کم | متوسط |
| تغییر در معماری | متوسط | کم |
| هزینه عملکرد | ناچیز | ناچیز |
| پیشنهاد | بله | فقط اگر راه اول ممکن نیست |

## نکات تکمیلی

- بعد از شکست `SaveChanges`، context را در حالت نامطمئن فرض کنید.
- پیام Outbox را باید در **همان تراکنش** و همان context با تغییر دامنه ذخیره کنید. در غیر این صورت الگوی Outbox معنایش را از دست می‌دهد.
- یک تست یکپارچه بنویسید: iteration اول را عمداً fail کنید و ثابت کنید iteration دوم هیچ اثری از اول ذخیره نمی‌کند.