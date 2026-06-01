---
title: "استفاده از FoxCloud روی Cloudflare Workers"
categories:
  - Web
tags:
  - web
  - vpn
  - vless
  - cloudflare
---

## مقدمه

اگر دنبال یک راه‌حل **بدون نیاز به سرور، بدون هزینه ماهانه و با سرعت بالا** برای دسترسی آزاد به اینترنت هستی، FoxCloud دقیقاً همان چیزی‌ست که باید بشناسیش. 

**FoxCloud** یک پروکسی‌سرور VLESS با کارایی بالاست که روی بستر Cloudflare Workers اجرا می‌شه و از زیرساخت Edge اون استفاده می‌کنه — یعنی کدت روی بیش از ۲۰۰ دیتاسنتر Cloudflare در سراسر دنیا اجرا می‌شه، نه روی یه سرور مرکزی. 

## FoxCloud چیست؟

FoxCloud یک پروکسی‌سرور VLESS مبتنی بر پروتکل WebSocket است که:

- **بدون هزینه** روی پلن رایگان Cloudflare کار می‌کنه 
- از پروتکل **VLESS** با انتقال WebSocket پشتیبانی می‌کنه 
- با تمامی کلاینت‌های **Xray** سازگار است 
- امنیت **TLS 1.3** را فراهم می‌کنه 
- سیستم **Subscription** (لینک اشتراک) برای مدیریت آسان کانفیگ‌ها دارد 

> ⚠️ **نکته مهم:** FoxCloud از برخی سرویس‌ها مثل توییتر و ChatGPT پشتیبانی نمی‌کند.  همچنین IP ثابت نیست و با هر اتصال ممکن است تغییر کند. 

## پیش‌نیازها

قبل از شروع نصب، به موارد زیر نیاز داری:

- یک حساب کاربری **Cloudflare** (رایگان) 
- **Node.js** نسخه ۱۸ به بالا (برای روش Build از سورس)
- آشنایی مختصر با خط فرمان

## معماری پروژه

```
foxcloud/
├── src/               # سورس TypeScript
├── scripts/           # اسکریپت‌های کمکی (UUID generator و...)
├── docs/              # مستندات کامل
├── wrangler.toml      # تنظیمات Cloudflare Worker
├── rolldown.config.js # تنظیمات Build
└── package.json
```


## متغیرهای محیطی (Environment Variables)

دو متغیر اصلی باید پیکربندی شوند: 

| متغیر | توضیح | مثال |
|-------|-------|------|
| `UUID` | لیست UUID کاربران (با کاما جدا) | `08dad8a6-...,49d598ee-...` |
| `PROXY_IP` | لیست IP‌های پروکسی با پورت | `172.66.45.9:443,104.18.128.25:443` |

## روش‌های نصب

### ✅ روش ۱: Deploy مستقیم (پیشنهادی)

سریع‌ترین روش برای راه‌اندازی: 

**مرحله اول — نصب Wrangler CLI:**
```bash
npm install -g wrangler
wrangler login
```

**مرحله دوم — دانلود فایل Build آماده:**

از صفحه [Releases پروژه](https://github.com/code3-dev/foxcloud/releases) آخرین فایل `worker.js` را دانلود کن.

**مرحله سوم — Deploy:**
```bash
wrangler deploy worker.js
```

### 🛠️ روش ۲: Build از سورس

برای کسانی که می‌خوان کنترل بیشتری روی کد داشته باشند: 

```bash
# کلون کردن ریپازیتوری
git clone https://github.com/code3-dev/foxcloud.git
cd foxcloud

# نصب وابستگی‌ها
npm install

# کپی فایل تنظیمات
cp wrangler.example.toml wrangler.toml
```

حالا فایل `wrangler.toml` را باز کن و مقادیر زیر را ویرایش کن:

```toml
name = "foxcloud"
main = "dist/worker.js"
compatibility_date = "2024-01-01"

[vars]
UUID = "YOUR-UUID-HERE"
PROXY_IP = "172.66.45.9:443,104.18.128.25:443"
```

سپس Build و Deploy کن:

```bash
npm run build
npm run deploy
```

### ⚙️ روش ۳: GitHub Actions (CI/CD)

برای دیپلوی اتوماتیک از طریق Git: 

1. ریپو را **Fork** کن
2. در تنظیمات ریپوی Fork شده، به بخش **Settings > Secrets and variables > Actions** برو
3. دو Secret زیر را اضافه کن:
   - `CLOUDFLARE_API_TOKEN` (از داشبورد Cloudflare بگیر)
   - `CLOUDFLARE_ACCOUNT_ID` (از صفحه Overview حسابت)
4. GitHub Actions را فعال کن
5. هر push به branch `master` به صورت خودکار Deploy می‌شه

## تولید UUID

برای هر کاربر باید یک UUID منحصربه‌فرد داشته باشی. روش‌های مختلف:

**در ویندوز (PowerShell):** 
```powershell
New-Guid
```

**با Node.js:**
```bash
node -e "console.log(require('crypto').randomUUID())"
```

**با اسکریپت FoxCloud:**
```bash
npm run generate-uuid
```

## پیکربندی از طریق داشبورد Cloudflare (بدون CLI)

اگر ترجیح می‌دی از رابط گرافیکی Cloudflare استفاده کنی:  

1. وارد [dash.cloudflare.com](https://dash.cloudflare.com) شو
2. از منوی **Compute & AI** وارد **Workers & Pages** شو
3. روی **Create application** کلیک کن، قالب **Hello World** را انتخاب کن
4. فایل `worker.js` را در ادیتور پیست کن و Deploy کن
5. در تب **Settings > Variables and Secrets**، متغیرهای `UUID` و `PROXY_IP` را اضافه کن

## استفاده از کانفیگ در کلاینت

پس از Deploy، آدرس Worker شما چیزی شبیه به این خواهد بود:  

```
https://your-worker-name.workers.dev
```

**لینک Subscription** برای ایمپورت به کلاینت‌ها:
```
https://your-worker-name.workers.dev/sub
```

این لینک را می‌تونی در کلاینت‌هایی مثل **Hiddify**، **v2rayNG** یا هر کلاینت سازگار با Xray ایمپورت کنی.  

## محیط توسعه و تست

```bash
# اجرای سرور توسعه (Local)
npm run dev

# اجرای تست‌ها
npm test
```


## محدودیت‌های پلن رایگان Cloudflare

| منبع | محدودیت رایگان |
|------|---------------|
| درخواست روزانه | ۱۰۰,۰۰۰ |
| مدت اجرا در هر درخواست | ۱۰ میلی‌ثانیه CPU |
| حافظه | ۱۲۸ MB |
| Sub-requests | ۵۰ در هر درخواست |

برای استفاده شخصی و ترافیک معمولی، پلن رایگان کاملاً کافیه.


> 📎 **لینک مخزن:** [github.com/code3-dev/foxcloud](https://github.com/code3-dev/foxcloud)  
> 📎 **لیست Proxy IP:** [github.com/code3-dev/code3-dev/blob/main/proxy_ip](https://github.com/code3-dev/code3-dev/blob/main/proxy_ip)