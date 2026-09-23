---
title: "PWA و Service Worker: از تاریخچه تا Offline First"
categories:
  - Web
tags:
  - web
  - pwa
  - service_worker
---

## بخش ۱: چرا اصلاً چیزی به اسم PWA به وجود آمد؟

قبل از این‌که وارد کد و API بشویم، باید مشکلی که این تکنولوژی حل می‌کند را دقیق بفهمید؛ وگرنه هر چه کد بنویسید فقط تقلید کورکورانه از یک تمپلیت است.

تا اوایل دهه ۲۰۱۰، یک شکاف بزرگ بین Web App و Native App وجود داشت. اپ‌های native سه امتیاز داشتند که وب نداشت:

- روی صفحه اصلی گوشی نصب می‌شدند (icon مستقل، بدون نوار آدرس مرورگر).
- بدون اینترنت هم کار می‌کردند.
- Push Notification داشتند و می‌توانستند کاربر را دوباره برگردانند.

در مقابل، وب یک مزیت بزرگ داشت که native هرگز نداشت: یک URL، بدون نصب، بدون App Store، قابل ایندکس در گوگل، و کراس-پلتفرم واقعی.

در ۱۵ ژوئن ۲۰۱۵، Alex Russell (مهندس تیم Chrome) و Frances Berriman (طراح) در مقاله‌ای با عنوان «Progressive Apps: Escaping Tabs Without Losing Our Soul» این دو دنیا را به هم وصل کردند و اسم «Progressive Web App» را رسمی کردند . نکته‌ی مهم این است که خودشان تأکید کردند این تکنولوژی‌ها از قبل وجود داشتند؛ کاری که آن‌ها کردند فقط نام‌گذاری یک الگوی جدید بود که به خاطر پیشرفت مرورگرها ممکن شده بود.

اما ستون فنی اصلی PWA، یعنی Service Worker، قبل‌تر از آن شروع شده بود. اولین commit های اسپک Service Worker را همان Alex Russell در فوریه ۲۰۱۳ نوشت، و اولین Public Working Draft در ۸ مه ۲۰۱۴ منتشر شد. Chrome از سال ۲۰۱۴ و Firefox از سال ۲۰۱۶ آن را ساپورت کردند. 

### مشکلی که قبل از Service Worker وجود داشت

قبل از Service Worker، یک API قدیمی‌تر به اسم `AppCache` (با فایل cache manifest) برای offline کردن سایت وجود داشت. مشکل AppCache این بود که declarative و غیرقابل‌کنترل بود: شما فقط یک لیست فایل به مرورگر می‌دادید و مرورگر خودش تصمیم می‌گرفت کِی و چطور آن‌ها را کش یا آپدیت کند. نتیجه این بود که توسعه‌دهنده‌ها دائم گیر «کش قدیمی که پاک نمی‌شود» می‌افتادند و AppCache در نهایت Deprecated شد. 

Service Worker این مشکل را با یک تغییر پارادایم حل کرد: به‌جای یک لیست ثابت، به شما یک اسکریپت جاوااسکریپت می‌دهد که مثل یک Programmable Network Proxy بین صفحه و شبکه می‌نشیند و شما با کد، دقیقاً کنترل می‌کنید چه درخواستی کش شود، از کجا سرو شود و کِی منقضی شود.

---

## بخش ۲: چهار ستون اصلی PWA

برای این‌که یک اپ واقعاً PWA حساب شود، این چهار مؤلفه باید کنار هم باشند:

- **HTTPS**: Service Worker فقط روی HTTPS کار می‌کند (به‌جز `localhost` برای توسعه)، چون این اسکریپت قدرت رهگیری همه‌ی ترافیک شبکه را دارد و روی HTTP ناامن، مسیر باز برای حملات Man-in-the-Middle می‌شود.
- **Web App Manifest**: یک فایل JSON که مرورگر را از هویت اپ (اسم، آیکون، رنگ، حالت نمایش) آگاه می‌کند تا بشود آن را نصب کرد.
- **Service Worker**: مغز اصلی offline-first، مسئول کش و رهگیری درخواست‌ها.
- **Responsive UI**: بدون طراحی واکنش‌گرا، تجربه «مثل اپ نیتیو» بی‌معنی است.

### Web App Manifest، دقیق‌تر

یک نمونه واقعی:

```json
{
  "name": "My Blog App",
  "short_name": "MyBlog",
  "start_url": "/index.html",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#0f172a",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

این فایل باید در `<head>` صفحه‌ی HTML اصلی این‌طور لینک شود:

```html
<link rel="manifest" href="/manifest.json">
```

نکته‌ی مهم درباره‌ی هر فیلد:

- **`start_url`**: مسیری که وقتی کاربر اپ را از هوم‌اسکرین باز می‌کند، لود می‌شود. این باید همیشه یک مسیر ثابت و قابل اعتماد باشد (مثلاً صفحه‌ی اصلی)، نه صفحه‌ای که کاربر از طریق یک لینک عمیق و موقتی وارد آن شده. اگر این را اشتباه تنظیم کنید، هر بار که کاربر آیکون را لمس می‌کند ممکن است در جای اشتباهی از اپ بیفتد.
- **`display`**: مقدار `standalone` نوار آدرس مرورگر و دکمه‌های ناوبری را حذف می‌کند و اپ را کاملاً شبیه native نشان می‌دهد. مقادیر دیگر شامل `fullscreen` (حتی status bar هم مخفی می‌شود، مناسب بازی‌ها) و `minimal-ui` (یک نوار کنترل حداقلی باقی می‌ماند) هستند.
- **`icons` با `purpose: "any maskable"`**: بعضی سیستم‌عامل‌ها (مثل اندروید) آیکون را در شکل‌های مختلف (دایره، مربع گرد) می‌برند. مقدار `maskable` به مرورگر می‌گوید که این آیکون طوری طراحی شده که در safe zone وسط تصویر، حتی بعد از برش، خراب نمی‌شود.
- **`background_color`**: رنگی است که در splash screen (قبل از لود شدن کامل اپ) نمایش داده می‌شود، نه پس‌زمینه‌ی دائمی اپ.

اشتباه رایج اینجا این است که توسعه‌دهنده‌ها فقط یک آیکون کوچک (مثلاً 192x192) می‌گذارند و آیکون بزرگ‌تر (512x512) را فراموش می‌کنند؛ نتیجه این می‌شود که splash screen روی گوشی‌های با صفحه‌ی بزرگ، آیکون بلور و پیکسلی نشان می‌دهد.

---

## بخش ۳: Service Worker - چرخه عمر واقعی

اینجا جایی است که اکثر توسعه‌دهنده‌ها گیج می‌شوند، چون Service Worker رفتار async و event-driven دارد و شبیه کد معمولی صفحه اجرا نمی‌شود. کد service worker در یک thread جدا از صفحه اجرا می‌شود، به DOM دسترسی ندارد، و ممکن است هر لحظه توسط مرورگر خاموش و دوباره روشن شود؛ پس نباید هیچ state مهمی را در متغیرهای global آن نگه دارید.

چرخه عمر پنج مرحله دارد:

1. **Register**: صفحه به مرورگر می‌گوید فایل service worker کجاست.
2. **Install**: مرورگر فایل را دانلود و نصب می‌کند. اینجا جایی است که Precaching (کش کردن فایل‌های اصلی App Shell) اتفاق می‌افتد.
3. **Waiting**: اگر یک نسخه‌ی قدیمی‌تر از service worker هنوز صفحات باز را کنترل می‌کند، نسخه‌ی جدید در حالت انتظار می‌ماند و فعال نمی‌شود.
4. **Activate**: وقتی همه‌ی تب‌های قدیمی بسته شوند، نسخه‌ی جدید فعال می‌شود. اینجا جای مناسب برای پاک کردن کش‌های قدیمی است.
5. **Fetch**: از این لحظه به بعد، هر درخواست شبکه‌ای از صفحات تحت کنترل، از event handler به اسم `fetch` عبور می‌کند و شما تصمیم می‌گیرید از کش سرو شود، از شبکه بیاید، یا ترکیبی از هر دو.

### مثال کامل: ثبت Service Worker

```javascript
// در فایل app.js که در صفحه اصلی لود می‌شود
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/'
      });
      console.log('Service Worker registered with scope:', registration.scope);
    } catch (error) {
      console.error('Service Worker registration failed:', error);
    }
  });
}
```

نکته‌ی `scope`: Service Worker فقط درخواست‌های زیر مسیری که در آن قرار دارد را کنترل می‌کند. اگر فایل `sw.js` را در `/app/sw.js` بگذارید، به‌صورت پیش‌فرض فقط `/app/` را کنترل می‌کند، نه کل سایت. این یکی از دام‌های رایج در پروژه‌های چند-بخشی است.

### مثال کامل: install و activate

```javascript
// sw.js
const CACHE_NAME = 'app-shell-v3';
const APP_SHELL_FILES = [
  '/',
  '/index.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/offline.html'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(APP_SHELL_FILES))
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) =>
      Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      )
    )
  );
});
```

چند نکته‌ی فنی مهم درباره‌ی این کد:

- **`event.waitUntil`**: اگر این را فراموش کنید، مرورگر ممکن است event `install` را «تمام‌شده» تصور کند در حالی که `cache.addAll` هنوز در حال دانلود فایل‌ها است، و service worker را زودتر از موعد فعال کند.
- **`cache.addAll`**: این متد یک عملیات atomic است؛ اگر حتی یکی از فایل‌های لیست ۴۰۴ برگرداند، کل عملیات install با شکست مواجه می‌شود و هیچ فایلی کش نمی‌شود. پس همیشه مسیرها را دقیق چک کنید.

نکته‌ی حیاتی: نام کش (`CACHE_NAME`) را هر بار که دیپلوی می‌کنید، ورژن کنید (`app-shell-v3`, `app-shell-v4`, ...). اگر این کار را نکنید، در event `activate` هیچ کش قدیمی‌ای شناسایی نمی‌شود و کاربر تا ابد نسخه‌ی قدیمی فایل‌ها را می‌بیند، حتی بعد از دیپلوی جدید. 

یک نکته‌ی دیگر که خیلی جاها نادیده گرفته می‌شود: به‌صورت پیش‌فرض، حتی وقتی service worker جدید فعال (`activated`) می‌شود، تب‌هایی که از قبل باز بودند همچنان توسط نسخه‌ی قدیمی کنترل می‌شوند تا رفرش شوند. اگر می‌خواهید نسخه‌ی جدید فوراً کنترل تب‌های باز را هم به دست بگیرد، باید در `activate` از `self.clients.claim()` استفاده کنید.

---

## بخش ۴: استراتژی‌های کش - قلب Offline-First

اینجا جایی است که تفاوت بین یک PWA حرفه‌ای و یک PWA باگ‌دار مشخص می‌شود. پنج استراتژی اصلی وجود دارد و انتخاب غلط بین آن‌ها باعث می‌شود کاربر یا محتوای قدیمی ببیند، یا اصلاً آفلاین کار نکند.

| استراتژی               | رفتار                                                                | بهترین کاربرد                                   |
| ---------------------- | -------------------------------------------------------------------- | ----------------------------------------------- |
| Cache First            | اول کش را چک کن، اگر بود همان را بده؛ اگر نبود از شبکه بگیر و کش کن  | فایل‌های static با نام versioned مثل CSS/JS/فونت |
| Network First          | اول از شبکه بگیر؛ اگر شبکه نبود، از کش بده                           | HTML و داده‌هایی که تازگی‌شان مهم است             |
| Stale While Revalidate | فوراً از کش جواب بده، همزمان در پس‌زمینه از شبکه بگیر و کش را آپدیت کن | محتوایی که سرعت مهم‌تر از تازگی مطلق است         |
| Cache Only             | فقط از کش، هرگز به شبکه نرو                                          | فایل‌های ثابتی که هرگز عوض نمی‌شوند               |
| Network Only           | فقط از شبکه، کش نادیده گرفته شود                                     | عملیات حساس مثل پرداخت                          |

### پیاده‌سازی Cache First

```javascript
self.addEventListener('fetch', (event) => {
  if (event.request.destination === 'style' || event.request.destination === 'script') {
    event.respondWith(
      caches.match(event.request).then((cachedResponse) => {
        if (cachedResponse) return cachedResponse;
        return fetch(event.request).then((networkResponse) => {
          return caches.open(CACHE_NAME).then((cache) => {
            cache.put(event.request, networkResponse.clone());
            return networkResponse;
          });
        });
      })
    );
  }
});
```

نکته‌ی تکنیکی: چرا `networkResponse.clone()`؟ چون یک شیء `Response` فقط یک بار قابل خواندن است (body آن یک stream است). اگر همان response را هم به `cache.put` بدهید و هم به عنوان خروجی fetch برگردانید، دومی با خطای «body already used» مواجه می‌شود.

### پیاده‌سازی Network First (برای HTML)

```javascript
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request)
        .then((networkResponse) => {
          const clone = networkResponse.clone();
          caches.open(CACHE_NAME).then((cache) => cache.put(event.request, clone));
          return networkResponse;
        })
        .catch(() => caches.match('/offline.html'))
    );
  }
});
```

### پیاده‌سازی Stale While Revalidate

```javascript
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/articles')) {
    event.respondWith(
      caches.open(CACHE_NAME).then(async (cache) => {
        const cachedResponse = await cache.match(event.request);
        const networkFetch = fetch(event.request).then((networkResponse) => {
          cache.put(event.request, networkResponse.clone());
          return networkResponse;
        });
        return cachedResponse || networkFetch;
      })
    );
  }
});
```

اینجا یک اشتباه رایج و خطرناک وجود دارد: کش کردن HTML به‌صورت Cache-First یا با TTL طولانی. HTML لینک به فایل‌های versioned دارد؛ اگر آن را کش دائمی کنید، کاربر حتی بعد از دیپلوی جدید، مارک‌آپ قدیمی می‌گیرد. همیشه برای HTML از Network-First یا Stale-While-Revalidate با max-age کوتاه استفاده کنید و Cache-First را فقط برای assetهای fingerprinted (که در نام فایلشان هش دارند) نگه دارید. 

یک نکته‌ی مهم درباره‌ی انتخاب استراتژی بر اساس `event.request.destination`: این فیلد به شما می‌گوید مرورگر این درخواست را برای چه چیزی می‌خواهد (`style`, `script`, `image`, `font`, و غیره)، و روش تمیزتری نسبت به چک کردن پسوند URL است چون به ساختار مسیر وابسته نیست.

---

## بخش ۵: اشتباهات رایجی که در پروداکشن باگ ایجاد می‌کنند

این بخش را جدی بگیرید، چون این‌ها دقیقاً همان چیزهایی هستند که در پروژه‌های واقعی، باگ‌های سخت و گاهی غیرقابل‌بازتولید ایجاد می‌کنند.

### کش کردن Response های Opaque بدون فیلتر

وقتی درخواست به یک منبع cross-origin بدون هدر CORS مناسب می‌رود (مثلاً یک فونت یا تصویر از CDN شخص ثالث)، مرورگر یک `Response` با status صفر (opaque) برمی‌گرداند. در این حالت جاوااسکریپت شما اجازه ندارد محتوای واقعی، status code، یا هدرهای آن پاسخ را ببیند؛ فقط می‌دانید که یک پاسخی دریافت شده است. مشکل این‌جاست: اگر آن سرور به هر دلیلی (خرابی موقت، rate limit، خطای شبکه) یک پاسخ خراب یا خالی برگرداند، شما به خاطر opaque بودن نمی‌توانید آن را تشخیص دهید، و اگر با استراتژی Cache-First آن را بدون فیلتر کش کنید، همان فایل خراب را برای همیشه به کاربر سرو می‌کنید.

راه‌حل استاندارد: فقط status code های `0` (opaque معتبر) و `200` (موفق واقعی) را کش‌پذیر در نظر بگیرید و بقیه را رد کنید. در Workbox این کار با پلاگین `CacheableResponsePlugin` انجام می‌شود .

### فراموش کردن Cache Versioning

اگر نام کش (`CACHE_NAME`) را در هر دیپلوی تغییر ندهید، مرحله‌ی `activate` هیچ کش قدیمی‌ای پیدا نمی‌کند تا پاک کند، و فایل‌های stale برای همیشه در Cache Storage باقی می‌مانند. این یکی از رایج‌ترین دلایل شکایت کاربران است که «بعد از آپدیت سایت، هنوز نسخه‌ی قدیمی می‌بینم».

### کش کردن بی‌رویه‌ی همه‌چیز

منطق «هر چه بیشتر کش کنیم بهتر است» غلط است. باید فقط App Shell حیاتی (HTML اصلی، CSS و JS پایه) را در مرحله‌ی install به‌صورت Precache نگه دارید و بقیه (تصاویر، داده‌های API) را با Runtime Caching (یعنی کش شدن هنگام درخواست واقعی) مدیریت کنید. در غیر این صورت حجم Cache Storage بی‌رویه رشد می‌کند و روی محدودیت فضای دیسک کاربر فشار می‌آورد، و مرورگر ممکن است خودش شروع به حذف داده‌های قدیمی کند بدون این‌که شما کنترلی روی آن داشته باشید.

### کش کردن داده‌ی API بدون سیاست تازگی

اگر پاسخ API را بدون استراتژی درست کش کنید (مثلاً موجودی انبار، قیمت، یا وضعیت حساب کاربری)، کاربر ممکن است اطلاعات قدیمی و گمراه‌کننده ببیند، حتی وقتی که آنلاین است. برای این نوع داده باید Network-First یا Stale-While-Revalidate با یک سقف زمانی مشخص استفاده کنید، نه Cache-First.

### گیج شدن با Scope

اگر service worker را در یک ساب‌مسیر ثبت کنید (مثلاً `/blog/sw.js`)، فقط همان مسیر (`/blog/*`) را کنترل می‌کند، نه کل دامنه. این یکی از دام‌های رایج در پروژه‌های چند-بخشی یا مونوریپو است که چند اپلیکیشن روی زیرمسیرهای یک دامنه دارند.

### تست کردن اشتباه در حین توسعه

تغییرات service worker در تب عادی مرورگر معمولاً به خاطر کش خود service worker نمایش داده نمی‌شود؛ مرورگر تا وقتی service worker فعلی «کنترل» می‌کند، نسخه‌ی جدید را در حالت waiting نگه می‌دارد. همیشه در حین توسعه از حالت Incognito یا از گزینه‌ی «Update on reload» در تب Application دوباره تست کنید .

### مشکل Mixed-Version Deploy

وقتی HTML جدید با JS یا CSS نسخه‌ی قدیمی (یا برعکس) قاطی می‌شود، به خاطر تایمینگ نامناسب service worker بین دو دیپلوی، باگ‌هایی به وجود می‌آید که دیباگ کردنشان بسیار سخت است چون فقط برای بعضی کاربران و به‌صورت متناوب رخ می‌دهد. راه‌حل استاندارد این است که برای assetها از فایل‌نام‌گذاری با هش محتوا (content hashing، مثل `app.a3f9c1.js`) استفاده کنید تا هر نسخه‌ی جدید یک URL کاملاً متفاوت داشته باشد و هیچ‌وقت با نسخه‌ی قدیمی قاطی نشود.

---

## بخش ۶: فراتر از کش فایل - داده و همگام‌سازی

Service Worker فقط برای کش کردن فایل استاتیک نیست. برای این‌که یک اپ واقعاً Offline-First باشد (نه فقط «صفحه‌اش بدون اینترنت باز می‌شود»)، دو تکنولوژی دیگر هم لازم است.

### IndexedDB

Cache API که در بخش‌های قبل دیدید، برای ذخیره‌ی جفت‌های `Request/Response` طراحی شده؛ خوب است برای فایل، ولی برای داده‌ی ساختاریافته و پویا مناسب نیست. **IndexedDB** یک دیتابیس NoSQL واقعی داخل مرورگر است که برای همین کار طراحی شده: مثلاً لیست پیام‌های یک اپ چت، سبد خرید، یا پیش‌نویس‌های یک ادیتور که کاربر باید بتواند بدون اینترنت هم آن‌ها را ببیند و ویرایش کند.

نمونه‌ی ساده‌ی باز کردن دیتابیس و ذخیره‌ی داده:

```javascript
function openAppDatabase() {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('AppDatabase', 1);

    request.onupgradeneeded = (event) => {
      const db = event.target.result;
      if (!db.objectStoreNames.contains('draftPosts')) {
        db.createObjectStore('draftPosts', { keyPath: 'id', autoIncrement: true });
      }
    };

    request.onsuccess = (event) => resolve(event.target.result);
    request.onerror = (event) => reject(event.target.error);
  });
}

async function saveDraftPost(content) {
  const db = await openAppDatabase();
  return new Promise((resolve, reject) => {
    const tx = db.transaction('draftPosts', 'readwrite');
    const store = tx.objectStore('draftPosts');
    const request = store.add({ content, createdAt: Date.now() });
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

نکته‌ی مهم: IndexedDB یک API کاملاً async و مبتنی بر event است، نه Promise-based به‌صورت native. به همین دلیل کتابخانه‌هایی مثل `idb` (نوشته‌ی Jake Archibald از تیم Chrome) محبوب هستند؛ آن‌ها همان API را با یک wrapper مبتنی بر Promise ساده‌تر می‌کنند، بدون این‌که رفتار زیرین را تغییر دهند.

### Background Sync API

مشکلی که این API حل می‌کند این است: فرض کنید کاربر آفلاین است و فرمی را ارسال می‌کند (مثلاً یک کامنت جدید). بدون Background Sync، آن درخواست fetch فقط fail می‌شود و از بین می‌رود، مگر این‌که خودتان با دست آن را در صف نگه دارید و منتظر رویداد `online` بمانید. Background Sync این کار را به مرورگر می‌سپارد: به مرورگر می‌گویید «وقتی اتصال برگشت، این tag را به من در service worker خبر بده»، و مرورگر خودش تشخیص می‌دهد شبکه دوباره وصل شده، حتی اگر تب کاربر بسته شده باشد.

```javascript
// در صفحه، وقتی فرم آفلاین ارسال می‌شود
async function submitPostOffline(postData) {
  await saveDraftPost(postData);
  const registration = await navigator.serviceWorker.ready;
  await registration.sync.register('sync-new-post');
}
```

```javascript
// در sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-new-post') {
    event.waitUntil(sendQueuedPostsFromIndexedDB());
  }
});

async function sendQueuedPostsFromIndexedDB() {
  const db = await openAppDatabase();
  const tx = db.transaction('draftPosts', 'readonly');
  const posts = await tx.objectStore('draftPosts').getAll();

  for (const post of posts) {
    await fetch('/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(post)
    });
  }
}
```

یک نکته‌ی مهم درباره‌ی پشتیبانی مرورگرها: Background Sync API به‌صورت استاندارد فقط در Chromium-based (Chrome, Edge) پشتیبانی می‌شود؛ Safari و Firefox آن را ساپورت نمی‌کنند. پس برای پروژه‌ای که باید روی همه‌ی مرورگرها کار کند، باید یک fallback دستی (چک کردن رویداد `online` در صفحه) هم بنویسید تا تجربه‌ی یکسانی داشته باشید.

الگوی کلی‌ای که این دو تکنولوژی با هم می‌سازند را **"Offline Queue Pattern"** می‌نامند: داده در IndexedDB نگه داشته می‌شود تا زمانی که شبکه وصل شود، و در آن لحظه Background Sync آن را خودکار ارسال می‌کند، بدون این‌که کاربر مجبور باشد دستی رفرش بزند یا دوباره فرم را پر کند.

---

## بخش ۷: ابزار دیباگ و چرا نباید همه‌چیز را دستی بنویسید

### دیباگ با Chrome DevTools

تب **Application** در Chrome DevTools مهم‌ترین ابزار شما برای کار با Service Worker است:

- بخش **Service Workers**: وضعیت فعلی service worker (`activated`, `waiting`, `redundant`) را نشان می‌دهد. دو گزینه‌ی حیاتی اینجا هست: **Update on reload** که هر بار صفحه را رفرش می‌کنید، نسخه‌ی جدید service worker را فوراً فعال می‌کند (بدون این‌که منتظر بسته شدن همه‌ی تب‌ها بمانید)، و **Bypass for network** که کلاً service worker را در حین رفرش نادیده می‌گیرد تا رفتار سایت بدون کش را ببینید.
- بخش **Cache Storage**: به شما اجازه می‌دهد محتوای دقیق هر کش را باز کنید، ببینید چه فایل‌هایی داخلش هست، و به‌صورت دستی هرکدام را حذف کنید. وقتی مشکلی مثل «کاربر نسخه‌ی قدیمی می‌بیند» دارید، اول سراغ اینجا بروید.
- بخش **IndexedDB**: محتوای دیتابیس‌های محلی را نشان می‌دهد، برای دیباگ کردن Offline Queue که در بخش قبل ساختیم مفید است.

یک تکنیک عملی: در حین توسعه، همیشه در **Incognito** تست کنید یا هر بار Cache Storage و Service Worker را دستی از DevTools پاک کنید. چون تغییرات service worker در تب عادی، به خاطر رفتار `waiting` که در بخش ۳ توضیح دادم، معمولاً بلافاصله دیده نمی‌شود و باعث سردرگمی کاذب می‌شود. 

### Audit با Lighthouse

ابزار **Lighthouse** (داخل همان DevTools، یا به‌صورت CLI) یک audit کامل PWA انجام می‌دهد و این موارد را چک می‌کند:

- آیا manifest معتبر است و آیکون‌های لازم را دارد؟
- آیا service worker ثبت شده و صفحه‌ی offline fallback دارد؟
- آیا سایت روی HTTPS سرو می‌شود؟
- آیا اپ قابل نصب (installable) است؟

نتیجه یک نمره‌ی عددی است که مشخص می‌کند کجای پیاده‌سازی ناقص مانده.

### چرا از صفر Vanilla Service Worker ننویسید

برای پروژه‌های واقعی، نوشتن دستی همه‌ی منطقی که در بخش‌های ۴ و ۵ دیدیم (استراتژی‌های کش، فیلتر کردن Opaque Response، Cache Versioning، پاکسازی در activate) خطرپذیر و پُر از جزئیات فراموش‌شدنی است. **Workbox**، کتابخانه‌ی رسمی گوگل، همین کارها را آماده و تست‌شده ارائه می‌دهد:

- استراتژی‌های Cache First، Network First، Stale While Revalidate را با یک خط کد فعال می‌کند.
- Cache Versioning را با Content Hashing خودکار مدیریت می‌کند، بدون این‌که خودتان نام کش را دستی عوض کنید.
- فیلتر کردن Response های Opaque را با پلاگین `CacheableResponsePlugin` انجام می‌دهد.
- محدود کردن حجم کش (Expiration Plugin) برای جلوگیری از رشد بی‌رویه‌ی Cache Storage دارد.

جمع‌بندی این بخش این است: یاد گرفتن Vanilla Service Worker (همان چیزی که در بخش‌های قبل با دست نوشتیم) برای فهم عمیق لایه‌ی زیرین ضروری است، اما پیاده‌سازی پروداکشن تقریباً همیشه باید روی Workbox ساخته شود، نه از صفر.

---

## Vanilla Service Worker چیست؟

اصطلاح «vanilla» در برنامه‌نویسی از آیس‌کریم وانیلی می‌آید؛ وانیلی همان طعم پایه و بدون هیچ افزودنی است. اصطلاح **Vanilla JS** اولین بار حدود سال ۲۰۱۲ محبوب شد (وبسایت شوخی‌آمیز vanilla-js.com توسط Eric Wastl، هرچند خودش می‌گوید این اصطلاح را او اختراع نکرده، فقط رایج‌ترش کرده) و به معنی نوشتن جاوااسکریپت خالص و استاندارد، بدون هیچ کتابخانه یا فریم‌ورک اضافه مثل jQuery یا React است. 

بر همین اساس، **Vanilla Service Worker** یعنی نوشتن فایل `sw.js` با API های خام و استاندارد مرورگر (`self.addEventListener('install', ...)`, `caches.open`, `caches.match` و غیره) بدون استفاده از هیچ کتابخانه‌ی کمکی مثل Workbox. دقیقاً همان کدهایی که در بخش‌های ۳ و ۴ این آموزش با دست نوشتیم (install، activate، fetch با استراتژی‌های کش) نمونه‌ی Vanilla Service Worker هستند.

### تفاوت با Workbox

Workbox یک لایه‌ی انتزاعی روی همان API های خام (Service Worker API و Cache Storage API) است؛ همان کارها را انجام می‌دهد اما با رابطه‌ای ساده‌تر و کمتر خطرپذیر. مثلاً همین کد Cache-First که در بخش ۴ با دست نوشتیم: 

```javascript
// Vanilla
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => cached || fetch(event.request))
  );
});
```

با Workbox این‌طور می‌شود:

```javascript
// با Workbox
import { registerRoute } from 'workbox-routing';
import { CacheFirst } from 'workbox-strategies';

registerRoute(
  ({ request }) => request.destination === 'style' || request.destination === 'script',
  new CacheFirst({ cacheName: 'static-assets' })
);
```

---

## بخش تکمیلی: پیاده‌سازی در Vue.js

همه‌ی مفهوم‌هایی که در بخش‌های قبل (Register، Install، Activate، Fetch، استراتژی‌های کش) یاد گرفتید، در Vue تغییر نمی‌کنند؛ فقط ابزار ساخت (build tool) خودش این کارها را خودکار می‌کند تا مجبور نباشید `sw.js` را کامل با دست بنویسید. برای پروژه‌های مدرن Vue که با **Vite** ساخته می‌شوند (که امروز استاندارد است، نه Webpack)، ابزار رسمی و پیشنهادی `vite-plugin-pwa` است. 

### نصب و پیکربندی

```bash
npm install -D vite-plugin-pwa
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    vue(),
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.svg', 'robots.txt'],
      devOptions: {
        enabled: true
      },
      manifest: {
        name: 'My Blog App',
        short_name: 'MyBlog',
        start_url: '/',
        display: 'standalone',
        background_color: '#0f172a',
        theme_color: '#0f172a',
        icons: [
          { src: 'pwa-192x192.png', sizes: '192x192', type: 'image/png' },
          { src: 'pwa-512x512.png', sizes: '512x512', type: 'image/png', purpose: 'any maskable' }
        ]
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg}'],
        navigateFallback: '/offline.html'
      }
    })
  ]
});
```

نکته‌ی مهم: زیر همین یک پلاگین، خودِ `vite-plugin-pwa` هم فایل manifest که در بخش ۲ دیدیم را می‌سازد، و هم service worker را با **Workbox** (که در بخش ۷ توضیح دادیم) به‌صورت خودکار تولید می‌کند؛ یعنی دیگر لازم نیست دستی `install`/`activate`/`fetch` بنویسید. 

فیلد `registerType` دو حالت اصلی دارد که مستقیماً به بحث چرخه‌ی عمر service worker در بخش ۳ مرتبط است:

- `autoUpdate`: به‌محض این‌که نسخه‌ی جدید service worker در دسترس باشد، خودش فعال می‌شود و صفحه را reload می‌کند، بدون تعامل کاربر.
- `prompt`: به شما اجازه می‌دهد یک پیام «نسخه‌ی جدید موجود است» به کاربر نشان دهید و اجازه بدهید خودش تصمیم بگیرد چه وقت رفرش کند؛ برای اپلیکیشن‌هایی که میان‌ کار کاربر (مثل فرم پر کردن) نباید ناگهان reload بخورد، این گزینه امن‌تر است.

### ثبت Service Worker در main.ts

```typescript
// main.ts
import { createApp } from 'vue';
import App from './App.vue';
import { registerSW } from 'virtual:pwa-register';

const updateServiceWorker = registerSW({
  immediate: true,
  onNeedRefresh() {
    console.log('نسخه‌ی جدید موجود است');
  },
  onOfflineReady() {
    console.log('اپ برای استفاده‌ی آفلاین آماده است');
  }
});

createApp(App).mount('#app');
```

`onNeedRefresh` و `onOfflineReady` دو callback هستند که مستقیم به همان رویدادهای `waiting` و `activate` که در بخش ۳ دیدیم متصل‌اند؛ یعنی به شما اجازه می‌دهند در UI واقعی Vue (مثلاً یک toast یا snackbar) به کاربر خبر بدهید، به‌جای این‌که فقط در console لاگ بزنید.

### مسیر قدیمی‌تر: vue-cli-plugin-pwa

اگر پروژه‌ای دارید که هنوز روی Vue CLI (Webpack) است، به‌جای Vite همان کار را با این دستور انجام می‌دهید:

```bash
vue add pwa
```

این دستور به‌صورت خودکار یک فایل `registerServiceWorker.js` می‌سازد و پیکربندی مربوط به آن را در `vue.config.js` زیر کلید `pwa` قرار می‌دهد. زیرِ آن هم دقیقاً همان `workbox-webpack-plugin` است که در بخش ۷ درباره‌اش صحبت کردیم، فقط با یک لایه‌ی Webpack به‌جای Vite.  

---

### الگوریتم دقیق Update - چرا گاهی آپدیت‌تان دیر می‌رسد

مرورگر هر بار که یک navigation جدید به origin شما اتفاق می‌افتد، به‌صورت خودکار فایل `sw.js` را دوباره دانلود می‌کند و آن را **بایت‌به‌بایت** با نسخه‌ی فعلی مقایسه می‌کند؛ اگر حتی یک بایت فرق داشته باشد، آن را «نسخه‌ی جدید» در نظر می‌گیرد و وارد چرخه‌ی install می‌کند. اما یک نکته‌ی مهم اینجاست: اگر آخرین دانلود کمتر از ۲۴ ساعت پیش بوده، مرورگر ممکن است همان نسخه‌ی کش‌شده‌ی HTTP فایل `sw.js` را برگرداند، نه نسخه‌ی واقعاً تازه از سرور. به همین دلیل، سرور شما باید هدر `Cache-Control: no-cache` را دقیقاً روی مسیر `sw.js` تنظیم کند تا این فایل هرگز توسط لایه‌ی HTTP Cache معمولی نگه داشته نشود. 

نکته‌ی عملی: اگر service worker شما را وسط یک import شده (`importScripts`) نگه می‌دارید و فقط محتوای آن فایل فرعی را عوض می‌کنید، مرورگر متوجه تغییر نمی‌شود، چون فقط فایل اصلی `sw.js` را بایت‌به‌بایت چک می‌کند. این یکی از دلایل رایج «چرا آپدیتم اصلاً دیده نمی‌شود» است.

### skipWaiting و clients.claim - کنترل دقیق زمان‌بندی

در بخش ۳ گفتیم service worker جدید در حالت `waiting` می‌ماند تا همه‌ی تب‌های قدیمی بسته شوند. دو متد به شما اجازه می‌دهند این رفتار پیش‌فرض را دستی کنترل کنید:

```javascript
self.addEventListener('install', (event) => {
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim());
});
```

- `self.skipWaiting()`: به service worker جدید می‌گوید منتظر نماند و فوراً وارد فاز `activate` شود، حتی اگر تب‌های قدیمی هنوز باز باشند.
- `self.clients.claim()`: به service worker تازه‌فعال‌شده اجازه می‌دهد بلافاصله کنترل تب‌های باز موجود را هم به دست بگیرد، بدون این‌که کاربر مجبور باشد صفحه را رفرش کند.

این ترکیب قدرتمند است اما یک ریسک واقعی دارد: اگر HTML و JS فعلی صفحه‌ی باز، با نسخه‌ی جدیدی که service worker تازه شروع به سرو کردنش کرده هم‌خوان نباشند (دقیقاً همان مشکل Mixed-Version Deploy که در بخش ۵ گفتیم)، ممکن است مصرف‌کننده‌ی API در صفحه با پاسخ‌های ناسازگار مواجه شود. به همین دلیل، `skipWaiting` + `clients.claim` معمولاً باید همراه با Content Hashing روی assetها استفاده شود، نه به‌تنهایی.

### Navigation Preload - رفع یک ضعف عملکردی جدی

یک مشکل واقعی در معماری service worker این است: وقتی کاربر یک صفحه را navigate می‌کند، مرورگر باید اول service worker را «boot» کند (که خودش زمان می‌برد)، و تنها بعد از آن event `fetch` اجرا می‌شود و درخواست واقعی شروع می‌شود. این یعنی یک تأخیر اضافه‌ی غیرضروری قبل از شروع دانلود صفحه.

**Navigation Preload** این مشکل را حل می‌کند: به مرورگر می‌گویید هم‌زمان با boot شدن service worker، درخواست شبکه را هم به‌صورت موازی شروع کند. 

```javascript
self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      if (self.registration.navigationPreload) {
        await self.registration.navigationPreload.enable();
      }
    })()
  );
});

self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      (async () => {
        const preloadResponse = await event.preloadResponse;
        if (preloadResponse) return preloadResponse;
        return fetch(event.request);
      })()
    );
  }
});
```

این ویژگی باید در `activate` فعال شود، نه در `install`، چون تا وقتی service worker فعال نیست، اصلاً fetch event ای برایش رخ نمی‌دهد که بخواهد از preload استفاده کند. 

### Push API - معماری واقعی نوتیفیکیشن

Push Notification که در ابتدای این آموزش به‌عنوان یکی از دلایل اصلی پیدایش PWA گفتیم، دقیقاً روی همین service worker سوار می‌شود. نکته‌ی مهم معماری این است: **Push API** کاملاً مستقل از این است که تب اپ باز باشد یا حتی اصلاً لود شده باشد؛ سرور شما پیام را به یک push service (که خودِ مرورگر مدیریت می‌کند، نه شما) می‌فرستد، و مرورگر service worker شما را برای پردازش آن، حتی وقتی هیچ تب اپ باز نیست، بیدار می‌کند. 

```javascript
self.addEventListener('push', (event) => {
  const data = event.data.json();
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icons/icon-192.png'
    })
  );
});
```

نکته‌ی امنیتی: برای این‌که سرور بتواند پیام امن به push service بفرستد بدون این‌که هرکسی بتواند جای شما پیام جعلی بفرستد، باید از **VAPID** (کلید عمومی/خصوصی که سرور امضا می‌کند) استفاده کنید. این بخش از معماری، مسئولیت بک‌اند است، نه فقط service worker.

### نکته‌ی امنیتی که کمتر گفته می‌شود

چون service worker می‌تواند هر درخواستی از origin خودش را رهگیری کند، اگر مهاجمی بتواند حتی یک بار یک service worker مخرب را در یک مسیر از سایت شما ثبت کند (مثلاً از طریق یک آسیب‌پذیری XSS یا آپلود فایل کنترل‌نشده)، آن service worker می‌تواند برای مدت طولانی (تا وقتی که خودش را unregister کند) روی همه‌ی ترافیک آن مسیر بنشیند، حتی بعد از رفع باگ اصلی. به همین دلیل الزام HTTPS به‌تنهایی کافی نیست؛ باید مسیر ثبت service worker (`scope`) را تا حد امکان محدود و کنترل‌شده نگه دارید و از آپلود فایل کاربر در مسیرهایی که می‌توانند به‌عنوان `.js` سرو شوند اجتناب کنید.

### ۱. محدودیت‌های سخت مرورگرها (Storage Quota)

هر origin (ترکیب پروتکل + دامنه + پورت) یک سقف ذخیره‌سازی جداگانه دارد که مرورگرها برای Cache API و IndexedDB اعمال می‌کنند. اگر این سقف را پر کنید، مرورگر شروع به حذف داده‌های قدیمی می‌کند، و شما کنترلی روی این‌که کدام کش حذف شود ندارید. راه‌حل: برای assetها از Workbox `ExpirationPlugin` استفاده کنید تا تعداد یا حجم هر کش را محدود کنید، و برای IndexedDB خودتان منطق پاک‌سازی قدیمی‌ها را بنویسید.

### ۲. تفاوت `caches.open` و `cache.addAll` در خطا

یک نکته‌ی ریز اما مهم: اگر `cache.addAll` روی یک فایل ۴۰۴ یا ۵۰۳ بخورد، کل عملیات `install` با خطا fail می‌شود و هیچ فایلی کش نمی‌شود. راه‌حل استاندارد این است که فایل‌های حیاتی (مثل `/index.html`, `/styles/main.css`) را در `addAll` بگذارید و بقیه را با `cache.put` جداگانه و در `try/catch` اضافه کنید تا یک فایل خراب، کل نصب را خراب نکند.

### ۳. `fetch` در service worker vs صفحه

یک تفاوت ظریف اما مهم: `fetch()` در service worker همیشه از cache HTTP معمولی مرورگر هم می‌خواند، مگر این‌که `cache: 'no-store'` را در گزینه‌های fetch بدهید. یعنی اگر قبلاً مرورگر یک فایل را کش کرده باشد، حتی اگر در cache API شما نباشد، ممکن است `fetch` همان را برگرداند. این می‌تواند در دیباگ گیج‌کننده باشد.

### ۴. `event.waitUntil` و `event.respondWith` - چرا هر دو؟

- `event.waitUntil`: به مرورگر می‌گوید «این promise را صبر کن، حتی اگر event handler تمام شد». برای `install` و `activate` استفاده می‌شود تا مطمئن شوید کش‌کردن یا پاک‌سازی قبل از رفتن به مرحله‌ی بعد تمام شده.
- `event.respondWith`: به مرورگر می‌گوید «این promise را به‌عنوان پاسخ اصلی fetch استفاده کن». فقط در `fetch` کاربرد دارد.

اگر `respondWith` را فراموش کنید، مرورگر خودش مستقیم به شبکه می‌رود و منطق service worker شما نادیده گرفته می‌شود.

### ۵. `self.registration` vs `navigator.serviceWorker.ready`

- `self.registration`: داخل service worker، به registration فعلی اشاره دارد.
- `navigator.serviceWorker.ready`: در صفحه‌ی اصلی، یک Promise است که وقتی resolve می‌شود، service worker فعال و آماده‌ی کنترل است.

این تفاوت مهم است وقتی می‌خواهید مثلاً `skipWaiting` را از داخل صفحه تریگر کنید (با یک کلیک کاربر).

### ۶. پیام‌رسانی بین صفحه و service worker

برای سناریوهایی مثل «کاربر دکمه‌ی آپدیت را زد، حالا به service worker بگو skipWaiting کن»، از `postMessage` استفاده می‌شود:

```javascript
// در صفحه
const registration = await navigator.serviceWorker.ready;
registration.active.postMessage({ type: 'SKIP_WAITING' });
```

```javascript
// در sw.js
self.addEventListener('message', (event) => {
  if (event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});
```

### ۷. دیباگ در فایرفاکس و سافاری

همه‌ی قابلیت‌هایی که گفتیم در کروم و اج (Chromium) کامل هستند، اما در فایرفاکس و سافاری بعضی API ها محدود یا غایب‌اند:

- Background Sync: فقط در Chromium.
- Navigation Preload: در فایرفاکس پشتیبانی نمی‌شود.
- Push API: در سافاری فقط روی iOS با محدودیت‌های خاص کار می‌کند.

پس اگر باید روی همه‌ی مرورگرها کار کنید، برای این API ها fallback دستی بنویسید یا فقط روی کروم/اج تکیه کنید و بقیه را degrade gracefully مدیریت کنید.