---
title: "مشکل عدم نمایش داده‌های به‌روزرسانی شده در یک سرویس ذخیره‌سازی با قابلیت eventual consistency"
categories:
  - Database
tags:
  - database
  - eventual consistency
---

# Read-Your-Writes Consistency: وقتی کاربر داده‌ی خودش رو نمی‌بینه

کاربر پست می‌فرسته، صفحه رو Refresh می‌کنه ولی پستش نیست. یه ثانیه بعد ظاهر می‌شه.

این باگ نیست. این مشکل **Read-Your-Writes (RYW) Consistency** هست.

این مشکل در کتاب **Designing Data-Intensive Applications** (Martin Kleppmann, Chapter 5) به این صورت تعریف شده:

> *"After a user writes data, they should see their own write in subsequent reads — regardless of which replica serves the request."*

وقتی یه سیستم **Leader/Follower Replication** داره، Write به Leader می‌ره ولی Read ممکنه از Followerی بیاد که هنوز sync نشده. نتیجه؟ کاربر داده‌ی خودش رو نمی‌بینه.

## چهار راه‌حل اصلی

### ۱. همیشه از Leader بخونیم

ساده‌ترین راه‌حل و البته بدترین. این روش Leader رو Bottleneck می‌کنه، Followerها بی‌استفاده می‌مونن و در Multi-Device کار نمی‌کنه. اگه کاربر با موبایل بنویسه و با لپ‌تاپ بخونه، Session مشترکی وجود نداره.

### ۲. Time Window Routing

بعد از Write، برای ۶۰ ثانیه همون کاربر رو به Leader هدایت کن.

مشکل: اگر Replication Lag از ۶۰ ثانیه بیشتر شد، چی؟ باید Window رو dynamically بر اساس lag واقعی تنظیم کنی وگرنه همون مشکل برمی‌گرده.

### ۳. LSN-Based Routing ✅

Leader بعد از هر Write، یه **Log Sequence Number (LSN)** برمی‌گردونه. Read بعدی فقط به Replicaای می‌ره که LSN اون `>= lastWriteLSN` باشه. به جای زمان، از موقعیت واقعی Replication استفاده می‌کنه — دقیق‌ترین روش.

### ۴. Commit Token (روش Oracle BDB)

Leader یه Token تولید می‌کنه، Client اون رو نگه می‌داره و با هر Read ارسال می‌کنه. Replica چک می‌کنه که آیا به اون Transaction رسیده یا نه. این معادل LSN-Based Routing هست اما به صورت explicit token از سمت Client.

## پیاده‌سازی در .NET: Redis + LSN-Based Routing

ترکیب **Redis + LSN-Based Routing** بهترین تعادل بین دقت و Scalability رو می‌ده. بعد از هر Write، `commitPosition` رو با TTL در Redis ذخیره می‌کنیم. در Read، فقط Replicaای رو انتخاب می‌کنیم که به اون position رسیده.

```csharp
// --- Domain ---
public record WriteAcknowledgment(bool Success, long CommitPosition);

public record Message(Guid Id, string Content, DateTime CreatedAt);

public enum DbEndpoint { Leader, Follower }

// --- Infrastructure ---
public interface IReplicationTracker
{
    Task RecordWriteAsync(string userId, long commitPosition);
    Task<long?> GetLastWritePositionAsync(string userId);
}

public sealed class RedisReplicationTracker : IReplicationTracker
{
    private readonly IDatabase _redis;
    private static readonly TimeSpan Ttl = TimeSpan.FromSeconds(120);

    public RedisReplicationTracker(IConnectionMultiplexer redis)
        => _redis = redis.GetDatabase();

    public Task RecordWriteAsync(string userId, long commitPosition)
        => _redis.StringSetAsync($"ryw:{userId}", commitPosition, Ttl);

    public async Task<long?> GetLastWritePositionAsync(string userId)
    {
        var val = await _redis.StringGetAsync($"ryw:{userId}");
        return val.HasValue ? long.Parse(val!) : null;
    }
}

// --- Read Router ---
public interface IReadRouter
{
    DbReplica Resolve(long? requiredLsn);
}

public sealed class LsnReadRouter : IReadRouter
{
    private readonly IReadOnlyList<DbReplica> _replicas;
    private readonly DbReplica _leader;

    public LsnReadRouter(IReadOnlyList<DbReplica> replicas, DbReplica leader)
    {
        _replicas = replicas;
        _leader = leader;
    }

    public DbReplica Resolve(long? requiredLsn)
    {
        if (requiredLsn is null)
            return _replicas.MinBy(r => r.Load) ?? _leader;

        var caughtUp = _replicas
            .Where(r => r.CurrentLsn >= requiredLsn)
            .MinBy(r => r.Load);

        return caughtUp ?? _leader; // Fallback به Leader
    }
}

public sealed class DbReplica(string connectionString)
{
    public string ConnectionString { get; } = connectionString;
    public long CurrentLsn { get; set; }
    public int Load { get; set; }
}

// --- Application Layer ---
public sealed class MessageService
{
    private readonly IMessageRepository _repo;
    private readonly IReplicationTracker _tracker;
    private readonly IReadRouter _router;

    public MessageService(
        IMessageRepository repo,
        IReplicationTracker tracker,
        IReadRouter router)
    {
        _repo = repo;
        _tracker = tracker;
        _router = router;
    }

    public async Task<WriteAcknowledgment> SendMessageAsync(string userId, string content)
    {
        var result = await _repo.WriteToLeaderAsync(content);

        if (result.Success)
            await _tracker.RecordWriteAsync(userId, result.CommitPosition);

        return result;
    }

    public async Task<IEnumerable<Message>> GetMessagesAsync(string userId)
    {
        var lastPosition = await _tracker.GetLastWritePositionAsync(userId);
        var replica = _router.Resolve(lastPosition);
        return await _repo.ReadFromAsync(replica);
    }
}

// --- Repository Interface ---
public interface IMessageRepository
{
    Task<WriteAcknowledgment> WriteToLeaderAsync(string content);
    Task<IEnumerable<Message>> ReadFromAsync(DbReplica replica);
}
```

### چند نکته مهم در این پیاده‌سازی

- **TTL روی Redis Key:** بعد از ۱۲۰ ثانیه، فرض می‌کنیم Replication کامل شده و Read دوباره به Follower می‌ره. این مقدار باید بر اساس میانگین Lag واقعی سیستم تنظیم بشه.
- **Fallback به Leader:** اگر هیچ Replicaای به LSN مورد نیاز نرسیده باشه، به Leader می‌ریم. این یعنی در بدترین حالت، مثل حالت اول عمل می‌کنیم — نه اینکه داده‌ی اشتباه بدیم.
- **`MinBy(r => r.Load)`:** بین Replicaهایی که catch-up کردن، اون با کمترین Load رو انتخاب می‌کنیم تا توزیع بار حفظ بشه.

## مقایسه استراتژی‌ها

| Strategy | دقت | Scalability | پیچیدگی |
|---|---|---|---|
| Always Leader | بالا | ضعیف ❌ | کم |
| Time Window | متوسط | خوب | کم |
| LSN-Based | بالا | عالی ✅ | متوسط |
| Sticky Session | متوسط | متوسط | کم |
| Commit Token | بالا | عالی ✅ | زیاد |

اگه این مشکل رو داری نادیده می‌گیری، کاربرهات دارن این تجربه رو می‌کنن — و فکر می‌کنن bug هست.

پس اگه سیستمی با تعداد یوزر زیاد و همزمانی زیاد داری، بهتره حواست به این مورد باشه. انتخاب بین این استراتژی‌ها به **میانگین Replication Lag**، **تعداد Replicaها** و **آیا Multi-Device داری یا نه** بستگی داره.