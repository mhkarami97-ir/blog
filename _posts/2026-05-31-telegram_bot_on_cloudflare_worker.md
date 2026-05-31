---
title: "انتشار بات تلگرام روی کلودفلر ورکر"
categories:
  - Web
tags:
  - web
  - bot
  - telegram
  - cloudflare
---

## انتشار بات تلگرام روی Cloudflare Worker
برای پیاده‌سازی ربات تلگرام بدون نیاز به سرور اختصاصی، Cloudflare Worker گزینه‌ای مناسب و رایگان محسوب می‌شود. این مستند مراحل ساخت و انتشار یک بات تلگرام روی Cloudflare Worker را به‌صورت گام‌به‌گام شرح می‌دهد.  

برای آشنایی بیشتر با Cloudflare Worker، مستندات رسمی آن در دسترس است:
[Cloudflare Worker Documentation](https://developers.cloudflare.com/workers/)

## پیش‌نیازها
پیش از شروع، ضروری است یک بات تلگرام ایجاد شده و توکن آن دریافت شود. برای این منظور باید در تلگرام با **BotFather** ارتباط برقرار کرده و دستور `/newbot` را ارسال کرد. پس از تکمیل مراحل، توکن ربات صادر می‌شود که در مراحل بعدی مورد استفاده قرار خواهد گرفت. [github](https://github.com/cvzi/telegram-bot-cloudflare)

## ایجاد پروژه
برای ایجاد یک پروژه جدید، دستور زیر در محیط ترمینال اجرا می‌شود:  

```bash
npm create cloudflare@latest my-telegram-bot
```

سپس وابستگی‌های پروژه نصب می‌گردند:  

```bash
npm install
```
پس از این مرحله، پایه پروژه آماده است و منطق ربات می‌تواند پیاده‌سازی شود.  
نمونه‌ای از سورس یک ربات آب‌وهوا جهت مرجع در لینک زیر موجود است:  
[Telegram Bot Example](https://github.com/mhkarami97/weather-bot)

## ذخیره‌سازی مقادیر محرمانه
پس از پیاده‌سازی کامل منطق ربات، توکن تلگرام و سایر مقادیر حساس باید به‌صورت Secret در Cloudflare Worker ذخیره شوند. برای این منظور دستورات زیر اجرا می‌شوند: [github](https://github.com/crazypeace/telegram-bot-on-worker)

```bash
npx wrangler secret put TELEGRAM_BOT_TOKEN
```

در اولین اجرا، مرورگر برای احراز هویت در حساب Cloudflare باز می‌شود. در ادامه، مقدار توکن در کنسول وارد می‌گردد. چنانچه Worker با نامی که در فایل `wrangler.toml` تعریف شده وجود نداشته باشد، یک Worker جدید ایجاد شده و کلید مذکور به‌عنوان Secret در آن ذخیره می‌شود. [github](https://github.com/crazypeace/telegram-bot-on-worker)  

در صورت بروز خطای TLS، دستور زیر پیش از تکرار دستور قبلی اجرا می‌شود:  

```bash
$env:NODE_TLS_REJECT_UNAUTHORIZED = "0"
```

در صورت استفاده از نمونه سورس فوق، دو Secret زیر نیز باید تنظیم شوند. این مقادیر به‌منظور افزایش امنیت و محدودسازی دسترسی به سرویس‌ها تعریف می‌شوند. مقدار هر کدام می‌تواند یک GUID یا هر رشته تصادفی دیگری باشد:  

```bash
npx wrangler secret put ADMIN_TOKEN
```

```bash
npx wrangler secret put TELEGRAM_WEBHOOK_SECRET
```

## انتشار Worker
پس از اتمام پیکربندی، دستور زیر برای انتشار بات روی Cloudflare Worker اجرا می‌شود:  

```bash
npm run deploy
```

## تنظیم Webhook
برای آنکه تلگرام بتواند با ربات ارتباط برقرار کند، Webhook باید پیکربندی شود. دستور زیر این کار را انجام می‌دهد؛ در آن `<your-worker>` با نام Worker و `<admin-token>` با مقدار Secret مربوط به `ADMIN_TOKEN` جایگزین می‌شود:  

```bash
curl.exe -X POST "https://<your-worker>.workers.dev/set-webhook" `
  -H "Authorization: Bearer <admin-token>"
```

دریافت پاسخ `ok` نشان‌دهنده موفقیت‌آمیز بودن تمام مراحل است. پس از این مرحله، ربات در تلگرام قابل استفاده خواهد بود.