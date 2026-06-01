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


---
---

# CFnew v2.9.8 — راهنمای کامل

**CFnew** یک پروژه Worker-based proxy است که چند پروتکل (VLESS، Trojan، xhttp) را روی Cloudflare Workers/Pages/Snippets اجرا می‌کند و رابط گرافیکی مدیریت دارد.   

## ⚠️ نکته حیاتی پیش از شروع

**Compatibility Date** حتماً باید روی `2026-01-20` باشد، وگرنه Worker کار نمی‌کند. این مورد را در مراحل زیر توضیح می‌دهم.   

همچنین برای رابط گرافیکی (KV) به یک **Custom Domain** یا حداقل Worker subdomain نیاز دارید. Workers Free tier کافی است.

## مرحله ۱ — تهیه UUID

```powershell
# PowerShell (Windows 11)
[guid]::NewGuid().ToString()
# مثال خروجی: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

این UUID را جایی ذخیره کنید — هم کلید ورود به پنل مدیریت است، هم UUID پروتکل VLESS.

## مرحله ۲ — روش‌های Deploy

سه روش وجود دارد. **Workers** را توصیه می‌کنم چون ساده‌ترین است.

### روش A: Workers (توصیه‌شده)

**1. دریافت فایل Worker:**

```bash
# دانلود فایل script اصلی (نسخه明文/plaintext)
curl -L "https://github.com/byJoey/cfnew/raw/main/%E6%98%8E%E6%96%87%E6%BA%90%E5%90%97" -o worker.js
```

یا اگر می‌خواهید نسخه obfuscated (برای دور زدن تشخیص Cloudflare) را بگیرید:

```bash
# نسخه obfuscated — رفتار کاملاً یکسان، فقط کد مبهم‌شده
curl -L "https://github.com/byJoey/cfnew/raw/main/%E5%B0%91%E5%B9%B4%E4%BD%A0%E7%9B%B8%E4%BF%A1%E5%85%89%E5%90%97" -o worker.js
```

**2. ایجاد Worker در Cloudflare Dashboard:**

- وارد [dash.cloudflare.com](https://dash.cloudflare.com) شوید
- از منوی چپ: **Workers & Pages** → **Create** → **Create Worker**
- نامی بدهید (مثلاً `cfnew-proxy`) → **Deploy**
- روی **Edit Code** کلیک کنید
- محتوای `worker.js` را paste کنید → **Deploy**

**3. تنظیم Compatibility Date (حیاتی):**

- روی Worker خود کلیک کنید → **Settings** → **Runtime**
- **Compatibility date** را روی `2026-01-20` بگذارید → **Save**   

### روش B: Pages (برای دامنه اختصاصی‌تر)

**1. در Cloudflare Dashboard:**
- **Workers & Pages** → **Create** → **Pages** → **Upload assets**
- یک فولدر بسازید، `worker.js` را به عنوان `_worker.js` در آن بگذارید
- فولدر را zip کنید و upload کنید → **Deploy**

**2. Compatibility Date برای Pages:**
- **Workers & Pages** → پروژه Pages → **Settings** → **Runtime** → تاریخ را `2026-01-20` بگذارید → **Save**   

### روش C: Snippets (دائمی‌ترین روش)

برای این روش حتماً به یک دامنه ثبت‌شده در Cloudflare نیاز دارید.   

- **Websites** → دامنه خود → **Snippets** → **Create Snippet**
- کد worker.js را paste کنید
- Rule اضافه کنید: `hostname: yourdomain.com/*`

## مرحله ۳ — تنظیم Environment Variables

بعد از deploy، متغیرهای زیر را اضافه کنید:

**Workers & Pages** → Worker → **Settings** → **Variables and Secrets** → **Add**

### متغیرهای اجباری

| متغیر | مثال | توضیح |
|---|---|---|
| `u` | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` | **اجباری** — UUID شما |

### متغیرهای اختیاری پرکاربرد

| متغیر | مقدار | توضیح |
|---|---|---|
| `p` | `172.66.45.9:443` | ProxyIP — آدرس relay |
| `d` | `/mysecretpath` | مسیر سفارشی به‌جای UUID |
| `wk` | `HK` یا `SG` یا `JP` | منطقه Worker (با `p` نمی‌شود همزمان داشت) |
| `ev` | `yes` | فعال‌سازی VLESS (پیش‌فرض: yes) |
| `et` | `yes` | فعال‌سازی Trojan (پیش‌فرض: no) |
| `ex` | `yes` | فعال‌سازی xhttp (پیش‌فرض: no) |
| `egi` | `no` | اگر `no` باشد، IP‌های default GitHub خاموش می‌شوند |

> **نکته معماری**: متغیر `p` و `wk` با هم **mutually exclusive** هستند — اگر `p` تنظیم کنید، `wk` نادیده گرفته می‌شود.   

## مرحله ۴ — تنظیم KV (رابط گرافیکی)

KV فضای ذخیره‌سازی Key-Value در Cloudflare است. با آن می‌توانید بدون redeploy تنظیمات را تغییر دهید.

**1. ایجاد KV Namespace:**
- **Workers & Pages** → **KV** → **Create a Namespace**
- نام بگذارید (مثلاً `cfnew-config`) → **Add**

**2. Bind به Worker:**
- Worker → **Settings** → **Bindings** → **Add** → **KV Namespace**
- Variable name: **`C`** (حرف بزرگ C — دقیقاً همین)   
- KV Namespace: `cfnew-config`
- **Save**، سپس Worker را دوباره deploy کنید

**3. دسترسی به پنل:**

مرورگر را باز کنید و بروید:
```
https://your-worker.workers.dev/{YOUR-UUID}
```

پنل گرافیکی مدیریت باز می‌شود. از اینجا می‌توانید:
- ProxyIP سفارشی اضافه کنید
- پروتکل‌ها را toggle کنید
- آدرس‌های بهینه را تست کنید
- ECH و ALPN را تنظیم کنید

## مرحله ۵ — دریافت Subscription Link

بعد از تنظیم، به آدرس زیر بروید تا لینک اشتراک بگیرید:

```
https://your-worker.workers.dev/{YOUR-UUID}
```

صفحه پنل باز می‌شود. لینک‌های subscription برای کلاینت‌های مختلف به‌صورت خودکار بر اساس User-Agent برمی‌گردد.

یا مستقیم:
```
# V2Ray/Xray format
https://your-worker.workers.dev/{YOUR-UUID}?sub=v2ray

# Clash format
https://your-worker.workers.dev/{YOUR-UUID}?sub=clash

# Sing-box format
https://your-worker.workers.dev/{YOUR-UUID}?sub=singbox
```

> در CFnew v2.9.8 دیگر به سرویس خارجی sub-converter نیاز نیست — تبدیل Clash/Sing-box/Surge/Loon/Quan X داخلی است.  

## مرحله ۶ — تنظیم کلاینت (Manual)

اگر بخواهید بدون subscription link، دستی اضافه کنید:

| فیلد | مقدار |
|---|---|
| **Protocol** | VLESS |
| **Address** | `your-worker.workers.dev` |
| **Port** | `443` |
| **UUID** | UUID شما |
| **Encryption** | `none` |
| **Transport** | `ws` |
| **Path** | `/{YOUR-UUID}` یا مسیر سفارشی |
| **TLS** | `tls` |
| **SNI** | `your-worker.workers.dev` |

## مرحله ۷ — تست تاخیر و یافتن بهترین IP

CFnew یک ابزار تست تاخیر داخلی دارد:

1. به پنل بروید: `https://your-worker.workers.dev/{YOUR-UUID}`
2. بخش **تست تاخیر** را باز کنید
3. می‌توانید IP‌های Cloudflare را به‌صورت Random تولید کنید یا از URL خارجی import کنید
4. Thread count را بین ۵ تا ۲۰ تنظیم کنید
5. **Start Test** — نتایج بر اساس تاخیر مرتب می‌شوند
6. IP‌های بهتر را به لیست ProxyIP اضافه کنید   

**ابزار desktop برای یافتن IP بهتر:**

```
https://github.com/byJoey/yx-tools/releases
```

## مرحله ۸ — مدیریت از طریق API (اختیاری)

اول API را در پنل فعال کنید (متغیر `ae = yes`)، سپس:

```bash
# افزودن ProxyIP جدید
curl -X POST "https://your-worker.workers.dev/{UUID}/api/preferred-ips" \
  -H "Content-Type: application/json" \
  -d '{"ip": "172.66.45.9", "port": 443, "name": "HK Node"}'

# افزودن چند IP به‌صورت batch
curl -X POST "https://your-worker.workers.dev/{UUID}/api/preferred-ips" \
  -H "Content-Type: application/json" \
  -d '[
    {"ip": "172.66.45.9", "port": 443, "name": "Node 1"},
    {"ip": "104.18.128.25", "port": 443, "name": "Node 2"}
  ]'

# پاک کردن همه IP‌ها
curl -X DELETE "https://your-worker.workers.dev/{UUID}/api/preferred-ips" \
  -H "Content-Type: application/json" \
  -d '{"all": true}'
```

## تفاوت Workers vs Pages vs Snippets

| | **Workers** | **Pages** | **Snippets** |
|---|---|---|---|
| **سختی راه‌اندازی** | آسان | متوسط | نیاز به دامنه |
| **آدرس** | `*.workers.dev` | `*.pages.dev` | دامنه خودت |
| **ماندگاری** | وابسته به پلن | خوب | دائمی‌ترین |
| **Free tier** | 100K req/day | 100K req/day | محدود |
| **توصیه** | شروع و تست | استفاده روزمره | دائمی |

## نکات Security

- **UUID را در هیچ جای عمومی** (GitHub، Telegram) به‌اشتراک نگذارید — دسترسی به پنل مدیریت از طریق همان UUID است   
- از متغیر `d` برای تنظیم یک **مسیر سفارشی تصادفی** استفاده کنید تا URL پنل قابل حدس زدن نباشد
- اگر Worker شما عمومی است، `ae = yes` (API) را **فعال نکنید** مگر اینکه ضروری باشد
- نسخه obfuscated را برای دور زدن تشخیص Cloudflare استفاده کنید 