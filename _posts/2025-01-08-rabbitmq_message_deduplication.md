---
title: "جلوگیری از ارسال پیام تکراری در RabbitMQ"
categories:
  - Net
tags:
  - net
  - rabbitmq
  - queue
---

یکی از پلاگین‌های مفید برای ربیت افزونه زیر است که امکان ارسال پیام فقط یکبار را به شما می‌دهد.  

[rabbitmq-message-deduplication](https://github.com/noxdafox/rabbitmq-message-deduplication)  

با این کار پیام‌های تکراری ارسالی به ربیت بصورت اتومات پاک می‌شوند و به کاربر نهایی ارسال نمی‌شوند.  

این افزونه به دو صورت برای Exchange و برای Queue کار می‌کند.  
برای فعال‌سازی آن بعد از نصب افزونه بر روی سرور ربیت نیاز هست بصورت زیر عمل کنید:  

برای Exchange:  

```csharp
channel.ExchangeDeclare(
    exchange: "name",
    type: "x-message-deduplication",
    durable: true,
    autoDelete: false
);
```

برای Queue:  

```csharp
channel.QueueDeclare(
    queue: "name",
    durable: true,
    exclusive: true,
    autoDelete: false,
    arguments: new Dictionary<string, object>
    {
        { "x-message-deduplication", true }
    });
```

اکنون کافی است برای پیام ارسال خود Header زیر را تعریف کنید و مقدار آن را با مقدار یونیک خود بطور مثال شناسه دیتابیس پر کنید تا پیام‌های تکراری در سمت خود ربیت مدیریت شوند.  

```csharp
x-deduplication-header
```

همچنین با مقدار زیر می‌توانید تایم پیش‌فرضی که شناسه بالا برای مدیریت تکراری بودن ذخیره می‌شود را تغییر دهید:  

```csharp
x-cache-ttl
```