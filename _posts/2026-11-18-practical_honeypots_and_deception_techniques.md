---
title: "راهنمای جامع Honeypot و تکنیک‌های فریب Deception"
description: "ماتریس جامع تله‌گذاری در لایه‌های شبکه، زیرساخت، سرور، کلود، کانتینر، دیتابیس، بک‌اند و فرانت‌اند، به همراه راهنمای پیاده‌سازی گام‌به‌گام و تحلیل فارنزیک ۳ هانی‌پات کلیدی."
categories: [security, devsecops]
tags: [honeypot, deception-technology, network-security, canarytokens, aws-security, blue-team, devsecops, intrusion-detection]
---

در سیستم‌های پایش و کشف نفوذ سنتی (مانند `SIEM`، `IDS/IPS` و `WAF`)، بزرگ‌ترین چالش تیم‌های امنیتی **آلارم‌های کاذب (`False Positives`)** و خستگی ناشی از حجم سرسام‌آور لاگ‌ها است. تفکیک ترافیک عادی از حملات هدفمند کار بسیار پرهزینه‌ای است.

اما در استراتژی **فریب سایبری (`Cyber Deception`)**، قواعد بازی معکوس می‌شود: شما منابع، فایل‌ها یا پورت‌هایی تعریف می‌کنید که **هیچ کاربرد عملیاتی ندارند** و در نتیجه، **هرگونه تعامل با آن‌ها با ضریب اطمینان ۱۰۰٪ نشان‌دهنده یک حمله یا اسکن غیرمجاز است (`Zero False Positives`)**.

در ادامه، ابتدا **ماتریس جامع تله‌گذاری** را با تمرکز ویژه بر لایه **شبکه و زیرساخت** مرور می‌کنیم و سپس به سراغ **پیاده‌سازی عمیق و گام‌به‌گام ۳ هانی‌پات استراتژیک** (بکاپ S3، فایل `.env` کرنل و کلیدهای قناری) می‌رویم.

---

## ۱. ماتریس جامع تله‌گذاری در لایه‌های مختلف (The Deception Matrix)

این جدول جامع‌ترین راهنمای تله‌گذاری در لایه‌های هشت‌گانه معماری و زیرساخت است:

| لایه معماری | نوع تله (Honeypot Type) | سناریو و نحوه پیاده‌سازی عملی | ابزار / مکانیزم مانیتورینگ | اقدام و واکنش خودکار (Response) |
| :--- | :--- | :--- | :--- | :--- |
| **Network & Infra** | پورت‌های تله روی فایروال (`Honey Ports / Port Tiers`) | باز نگه داشتن چند پورت جذاب بدون سرویس (مثل پورت‌های `21`، `23`، `445`، `3389`) در فایروال سرور یا لبه شبکه با ابزاری مثل `Portsentry` یا قوانین `iptables`. | ماژول لاگ‌گیری فایروال / `Portsentry Daemon` | مسدودسازی درجا و اتوماتیک `IP` اسکنر در سطح لبه فایروال به محض اولین پکت `SYN`. |
| **Network & Infra** | روت‌های SNMP تله (`Decoy Community String`) | فعال‌سازی یک Community String خواندنی با نام گول‌زننده مثل `infra_monitor` یا `public_adm` روی سوییچ‌ها و روترها بدون ارجاع به MIBهای واقعی. | لاگ دسترسی `SNMP Trap` در سوییچ/سرور مانیتورینگ | هر کوئری به این نام نشان‌دهنده اسکنرهای کشف شبکه (`Network Reconnaissance`) است؛ اعلام هشدار با اولویت بالا. |
| **Network & Infra** | وب‌اینترفیس شبیه‌سازی شده روتر/VPN | راه‌اندازی یک صفحه لاگین فیک وب شبیه به پنل میکروتیک، سیسکو یا فایروال فورتی‌نت روی یک پورت غیرمعمول داخلی (مثلاً پورت `8443`). | سرور وب سبک دامی (پایتون/Go) با وب‌هوک اختصاصی | ثبت تلاش‌های ورود (`Brute Force`)، ثبت پسوردهای احتمالی سازمان و بلاک منبع حمله. |
| **Network & Infra** | تله حملات مسموم‌سازی ARP (`ARP Spoofing Trap`) | اجرای یک Daemon سبک در شبکه داخلی که تغییرات غیرمجاز در نگاشت `MAC-to-IP` سرورهای کلیدی (مانند Gateway یا DNS) را رصد کند. | ابزارهای مانیتورینگ مثل `Arpwatch` یا اسکریپت مانیتور ARP جدول لینوکس | کشف حملات `Man-in-the-Middle (MitM)` در لایه ۲؛ قرنطینه پورت سوییچ از طریق `802.1X` یا `Port Security`. |
| **Network & Infra** | رکوردهای DNS تله (`Honey DNS Records`) | ثبت رکوردهای دامنه‌ای ساختگی در سرور DNS داخلی مانند `vpn-internal.corp.local` یا `secret-vault.lan` اشاره‌کننده به یک IP مانیتور شده. | لاگ کوئری‌های DNS در `Bind9` یا `Windows DNS Server` | شناسایی هاست‌های آلوده درون شبکه که با اسکریپت‌های کشف دامنه (`Domain Enumeration`) به دنبال اهداف می‌گردند. |
| **Network & Infra** | شبکه وای‌فای فریبنده (`Decoy SSID / Honey AP`) | تعریف یک شبکه بی‌سیم مجزا با نامی مشابه سازمان (مثل `Corp_Secure_Internal`) با احراز هویت کنترل‌شده و ایزوله کامل. | لاگر اکسس‌پوینت / مانیتورینگ وای‌فای | کشف مهاجمان حاضر در نزدیکی فیزیکی سازمان یا تلاش‌های ساخت `Rogue AP` و `Evil Twin`. |
| **Network & Infra** | سرور کندکننده ارتباط (`Tarpit - Endlessh`) | راه‌اندازی سرویس `Endlessh` روی پورت ۲۲ اصلی (در حالی که پورت مدیریت SSH سرور به پورتی اختصاصی منتقل شده است). | لاگ سرویس `Endlessh` | کند کردن فرآیند اسکن اتوماتیک، قفل کردن نخ‌های پردازشی مهاجم با ارسال فوق‌العاده کند بنر SSH. |
| **Network & Infra** | پوشه شبکه اشتراکی فریبنده (`Honey SMB Share`) | ساخت یک SMB Share با نام وسوسه‌انگیز `\\fileserver\Finance_Q4_Confidential` با دسترسی `Everyone`، بدون فایل واقعی اما با فعال بودن لاگ دسترسی. | لاگ رویدادهای ویندوز (Event IDهای `4663` و `4656`) | اعلام نفوذ در شبکه داخلی و تلاش مهاجم برای جمع‌آوری داده‌ها از طریق پروتکل SMB. |
| **Active Directory** | اکانت‌های تله با `SPN` (`Honey SPN`) | ایجاد اکانت‌های کاربری بدون فعالیت مثل `svc_backup_admin` با مقدار `ServicePrincipalName` فعال و رمز عبور طولانی. | Event Logهای امنیتی ویندوز (ایونت `4769` درخواست تیکت TGS) | کشف حملات سرقت هش تیکت‌ها (`Kerberoasting`) در همان مرحله اولیه درخواست تیکت. |
| **Active Directory** | اکانت وسوسه‌انگیز Domain Admin | ساخت یک کاربر ساختگی با نام‌های استاندارد ادمین مثل `adm_infra_support` و عضویت صوری در گروه‌های حساس، بدون استفاده واقعی. | پایش تلاش‌های لاگین در کنترلر دامین (ایونت `4625` و `4624`) | نشانه قطعی نشت نام‌های کاربری یا حملات `Pass-the-Hash` در شبکه داخلی. |
| **Server & OS** | فایل جعلی `.bash_history` | درج دستورات ساختگی در فایل تاریخچه شل شامل دستورات اتصال `SSH` با کلید به سرورهای تله داخلی. | مانیتورینگ لاگ سرورهای مقصد | گمراه کردن مهاجم، کشاندن او به محیط‌های ایزوله و ثبت متدهای نفوذ و ابزارهای او. |
| **Server & OS** | کلیدهای SSH ساختگی | قرار دادن یک فایل کلید نامعتبر `~/.ssh/id_rsa` که در توضیحات آن نام سرورهای دیگر قید شده است. | لاگ سرویس `sshd` در شبکه | هشدار قطعی در صورت تلاش برای اتصال با این کلید؛ ایزوله کردن هاست مبدا. |
| **Server & OS** | جاب زمان‌بندی‌شده فریبنده (`Cron Trap`) | ایجاد فایلی ساختگی در `/etc/cron.d/sync_db_passwords` با سطح دسترسی خواندن برای کاربران سیستم. | مانیتورینگ سطح فایل با ساب‌سیستم `auditd` لینوکس | کشف کاربران یا نفوذگرانی که در مرحله ارتقای سطح دسترسی (`Privilege Escalation`) فایل‌های حساس را می‌خوانند. |
| **Server & OS** | داکیومنت‌های اداری قناری (`Word/Excel Token`) | قرار دادن فایل‌هایی با نام `employee_salaries_2026.docx` دارای توکن‌های `Canarytokens` در مسیرهای عمومی سرور. | سرور اعلان مرکزی `Canarytokens` | به محض باز شدن فایل، ایمیل هشدار با آدرس IP، سیستم‌عامل و نرم‌افزار بازکننده ارسال می‌شود. |
| **Backend & API** | روت‌های جذاب اسکنرها | تعریف اندپوینت‌های معروفی مثل `/.env`، `/.git/HEAD`، `/actuator/env`، `/wp-login.php`، `/.aws/credentials` در وب‌سرور یا کد بک‌اند. | لاگ فریم‌ورک / Nginx / WAF | مسدودسازی فوری IP در لایه فایروال (`Fail2ban` یا Cloudflare WAF API) بدون نیاز به پردازش در اپلیکیشن. |
| **Backend & API** | اندپوینت فیک مدیریت ادمین | ساخت روت‌هایی مثل `/api/v1/system/export-database` یا `/debug/pprof` که پاسخ `401 Unauthorized` می‌دهد اما هر تلاشی برای احراز هویت را ثبت می‌کند. | Middleware احراز هویت / لاگر امنیتی | مسدود کردن اکانت یا IP و لاگ‌گیری دقیق کلیدها و پسوردهایی که مهاجم برای شکستن رمز عبور استفاده می‌کند. |
| **Backend & API** | فیلدهای تله در GraphQL | افزودن فیلدها یا متاسیستم‌های ساختگی در اسکیما مثل `__debugInternalQuery` یا `systemSuperAdmin`. | Resolver لاگر در GraphQL Server | هر فراخوانی این کوئری نشان‌دهنده نفوذگر یا اسکنر تخصصی GraphQL است؛ نشست کاربر بلافاصله باطل می‌شود. |
| **Backend & API** | توکن جعلی JWT در کوکی | ارسال یک توکن JWT ساختگی در پاسخ‌ها با پیلود دیکودپذیر شامل `{"role": "superadmin", "canary_id": "jwt-01"}` بدون امضای واقعی. | گیت‌وی API یا میدل‌ور تأیید توکن | هکری که توکن را دستکاری یا استفاده کند، بلافاصله شناسایی و IP او قرنطینه می‌شود. |
| **Backend & API** | هدرهای ساختگی HTTP | ارسال هدرهای گول‌زننده در پاسخ سرور مانند:<br>`X-Internal-Backend-Ip: 10.10.50.12` | مانیتورینگ شبکه و فایروال روی IPهای فیک | اگر درخواستی به این IP داخلی غیرواقعی ارسال شود، نفوذ به لایه شبکه مشخص می‌شود. |
| **Database** | جدول تله دیتابیس (`Honey Table`) | ساخت جدولی با نام‌های فریبنده مثل `tbl_admin_credentials` یا `system_secrets` با داده‌های ساختگی. | تعریف دیتابیس `Trigger` روی رویداد `SELECT` یا لاگ‌های Audit دیتابیس | اثبات قطعی نفوذ از طریق حمله `SQL Injection`؛ لغو فوری کانکشن مهاجم و ارسال آلرت با بالاترین اولویت. |
| **Database** | رکورد تله در جدول کاربران (`Honey User`) | درج یک ردیف در جدول واقعی کاربران با ایمیلی مثل `emergency_superadmin@decoy.domain.com` با پسوردی غیرقابل حدس. | سیستم احراز هویت (`Auth Service`) | هر تلاشی برای ورود یا بازیابی رمز این اکانت به معنای نشت کامل دیتابیس یا نشت سورس‌کد است. |
| **Database** | رویه ذخیره‌شده فریبنده (`Honey Stored Proc`) | ایجاد پروسیجری مثل `sp_export_all_passwords` که در بدنه آن ارسال وب‌هوک به لاگر امنیتی تعبیه شده است. | موتور دیتابیس (SQL Server / PostgreSQL) | به دام انداختن مهاجم در مرحله حرکت جانبی درون دیتابیس. |
| **Frontend** | فیلد مخفی فرم (`Hidden Input`) | قرار دادن یک اینپوت با نام جذاب مثل `billing_account_no` یا `website_url` که با CSS کلاس `hidden` یا `opacity: 0` پنهان شده است. کاربران واقعی آن را پر نمی‌کنند اما بات‌ها پر می‌کنند. | اعتبارسنجی در کنترلر بک‌اند (`Request Validation`) | رد درخواست با کد `400/422`، اضافه کردن IP به لیست مسدودی موقت یا فعال‌سازی چالش کپچای سختگیرانه. |
| **Frontend** | اندپوینت فیک در باندل JS | قرار دادن کامنت یا رشته ساختگی در کدهای کامپایل‌شده فرانت:<br>`const INTERNAL_DEBUG_URL = "https://api.domain.com/v1/_internal_metrics";` | رصد روت در وب‌سرور یا فایروال (Nginx / Cloudflare) | مهاجم با اسکن کدهای جاوااسکریپت این روت را تست می‌کند. فوراً IP او به بلک‌لیست WAF اضافه می‌شود. |
| **Frontend** | لینک نامرئی خزش (`Decoy Crawl Link`) | افزودن تگ لینک نامرئی در انتهای صفحات وب:<br>`<a href="/admin-panel-backup" style="display:none" rel="nofollow">Admin</a>` | لاگ‌های دسترسی وب‌سرور یا روتینگ فریم‌ورک | شناسایی خودکار اسکرپرهای تهاجمی و ابزارهای اسکن آسیب‌پذیری وب؛ بلاک فوری ترافیک خزنده. |
| **Frontend** | کوکی یا LocalStorage تله | ذخیره کردن کلید ساختگی در مرورگر کاربر مثل:<br>`localStorage.setItem("debug_admin_token", "canary_xxx")` | مانیتورینگ ریدایرکت یا ارسال درخواست با هدر تله | اگر درخواستی با این مقدار به سرور فرستاده شود، نشان‌دهنده دستکاری دستی یا اکستنشن‌های مشکوک مرورگر است. |
| **Frontend** | فایل Source Map تله (`.js.map`) | قرار دادن فایل `app.bundle.js.map` ساختگی در وب‌سرور که در انتهای آن به یک URL لاگر یا وب‌هوک Canarytoken ارجاع داده شده است. | وب‌سرور یا سرور لاگ Canary | هکرها پس از دانلود نقشه سورس، بلافاصله آن را در DevTools لود می‌کنند؛ تله تریگر شده و آلارم ارسال می‌شود. |
| **Cloud & DevOps** | سکرت‌های فیک در مخازن Git | قرار دادن کلیدهای ساختگی AWS یا توکن‌های GitHub در فایل‌های `.env.example` یا کامیت‌های گذشته. | ابزارهای اسکن کلید و زیرساخت Canarytokens | آگاهی از نشت کد منبع، کلون شدن غیرمجاز ریپازیتوری یا هک شدن سیستم توسعه‌دهنده. |
| **Cloud & DevOps** | فایل فیک `terraform.tfstate` | قرار دادن یک استیت فایل بلااستفاده در یک باکت یا مخزن که خروجی متغیرهای آن حاوی توکن‌های دسترسی کلود باشد. | رصد فراخوانی متدها در لاگ‌های کلود | به دام انداختن هکرهایی که با ابزارهای خودکار در پی کشف زیرساخت ابری هستند. |
| **Cloud & DevOps** | باکت تله در کلود (`S3 Honey Bucket`) | ساخت باکت با نام جذاب عمومی مثل `company-internal-backups-prod` بدون محتوای حساس و با لاگینگ فعال. | `AWS CloudTrail` / `GCP Cloud Audit Logs` | تشخیص مهاجمانی که با ابزارهای `Bucket Enumeration` باکت‌های سازمان را اسکن می‌کنند. |
| **Cloud & DevOps** | کاربر IAM بدون دسترسی (`Honey IAM User`) | ساخت کاربر در AWS با نام `admin-deployer` و تولید Access Key برای آن بدون الصاق هیچ Policy معتبر. | سرویس `Amazon GuardDuty` / `CloudTrail` | هر تلاشی برای استفاده از این کلیدها نشان‌دهنده نشت فایل یا استخراج متغیرهای محیطی است. |
| **Container & K8s** | سکرت قناری در کوبرنتیز (`K8s Honey Secret`) | ایجاد سکرت در نیم‌اسپیس با نام `database-master-creds` حاوی کلیدهای قناری. | `Kubernetes Audit Logs` | هرگونه تلاش برای خواندن این سکرت توسط کانتینرهای آلوده یا پادهای مشکوک ثبت و پاد فوراً `Terminate` می‌شود. |
| **Container & K8s** | توکن `ServiceAccount` فیک | قراردادن توکن نامعتبر یا قناری در مسیر `/var/run/secrets/kubernetes.io/serviceaccount` داخل یک پاد آسیب‌پذیر. | `K8s API Server Audit` | کشف فوری سناریوهای فرار از کانتینر (`Container Escape`) یا تلاش برای جابه‌جایی افقی در کلاستر. |

---

## ۲. پیاده‌سازی گام‌به‌گام و عمیق ۳ هانی‌پات استراتژیک

در این بخش، ۳ تله کاربردی که در سناریوهای سرقت فایل و سکرت بالاترین بازدهی را دارند با جزییات کامل فنی، پیکربندی، فارنزیک و اسکریپت‌های اتوماسیون بررسی می‌شوند.

---

### هانی‌پات اول: بکاپ دیتابیس ساختگی در `AWS S3` (`prod-db-backup.sql.gz`)

#### ۱. منطق و جذابیت تله برای مهاجم:
هنگامی که مهاجم یک کانتینر، سرور EC2 یا ماشین توسعه را اکسپلویت می‌کند، معمولاً نقش انتساب‌یافته به آن منبع (مانند IAM Instance Profile) دسترسی‌هایی به S3 دارد. مهاجم ابتدا متد `aws s3 ls` را برای جستجوی داده‌های حساس اجرا می‌کند و با دیدن فایلی با نام `prod-db-backup.sql.gz` یا `users_export_dump.tar.gz` وسوسه می‌شود بلافاصله آن را با `aws s3 cp` دانلود کند.

#### ۲. پیاده‌سازی و پیکربندی در AWS:

* **مرحله الف: فعال‌سازی لاگینگ داده‌ای در CloudTrail:**
  به صورت پیش‌فرض، CloudTrail تنها لاگ‌های Management Events را ثبت می‌کند و رویدادهای خواندن آبجکت‌های S3 (`s3:GetObject`) را برای مدیریت هزینه ثبت نمی‌کند. باید لاگینگ رویدادهای داده‌ای (Data Events) را برای باکت یا مسیر مربوطه فعال کنید:

```bash
aws cloudtrail put-event-selectors --trail-name PrimarySecurityTrail \
  --event-selectors '[{
    "ReadWriteType": "ReadOnly",
    "IncludeManagementEvents": false,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::my-company-assets-prod/backups/prod-db-backup.sql.gz"]
    }]
  }]'
```

* **مرحله ب: رول رویدادمحور در `Amazon EventBridge`:**
  تنظیم یک EventBridge Rule که رخداد فراخوانی متد `GetObject` روی این فایل را مستقیماً به یک تاپیک SNS، اسلک وبهوک یا لامبدا وصل کند:

```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["s3.amazonaws.com"],
    "eventName": ["GetObject"],
    "requestParameters": {
      "bucketName": ["my-company-assets-prod"],
      "key": ["backups/prod-db-backup.sql.gz"]
    }
  }
}
```

#### ۳. تحلیل فارنزیک و بررسی رفتار مهاجم:
وقتی این هشدار به صدا درمی‌آید، در پیلود رخداد اطلاعات حیاتی زیر ثبت شده است:

```json
{
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAXXXXXXXXXXXX:i-0123456789abcdef0",
    "arn": "arn:aws:sts::123456789012:assumed-role/AppInstanceRole/i-0123456789abcdef0"
  },
  "sourceIPAddress": "198.51.100.45",
  "userAgent": "aws-cli/2.15.0 Python/3.11.6 Linux/5.15.0",
  "requestParameters": {
    "bucketName": "my-company-assets-prod",
    "key": "backups/prod-db-backup.sql.gz"
  }
}
```

* **ردگیری فعالیت‌های قبلی نقش مورد سوءاستفاده:**
  بلافاصله در CloudTrail دستور زیر را اجرا کنید تا بفهمید این هویت قبل از دانلود بکاپ چه کارهای دیگری انجام داده است:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=AppInstanceRole \
  --start-time "2026-11-18T00:00:00Z" \
  --query 'Events[*].[EventTime, EventName, SourceIPAddress, AccessKeyId]' \
  --output table
```

#### ۴. اتوماسیون پاسخ فوری (`Automated Quarantine Lambda`):
تابع سرورلس زیر بلافاصله به عنوان Target به EventBridge Rule متصل شده و یک `Inline Policy` مسدودکننده کامل (`DenyAll`) به نقش متصل می‌کند:

```python
import boto3
import json

iam = boto3.client('iam')

def lambda_handler(event, context):
    detail = event.get('detail', {})
    user_identity = detail.get('userIdentity', {})
    arn = user_identity.get('arn', '')
    
    # استخراج نام نقش IAM از ARN
    if "assumed-role" in arn:
        role_name = arn.split("/")[1]
        deny_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Deny",
                    "Action": "*",
                    "Resource": "*"
                }
            ]
        }
        iam.put_role_policy(
            RoleName=role_name,
            PolicyName="EMERGENCY_INCIDENT_LOCKDOWN",
            PolicyDocument=json.dumps(deny_policy)
        )
        print(f"Quarantined compromised role: {role_name}")
```

---

### هانی‌پات دوم: فایل `.env` تله با پایش کرنل لینوکس و قرنطینه شبکه

#### ۱. سناریو و جذابیت محتوا:
در لینوکس، مهاجم پس از دستیابی به شل با ابزارهایی مثل `linpeas` یا دستورات ساده `grep` و `find` فایل‌های حاوی رمز عبور را جستجو می‌کند. مسیرهایی نظیر `/home/ubuntu/.env`، `/var/www/.env` یا `/root/.env` اولین گزینه‌های بررسی هستند.

محتوای فایل باید کاملاً واقعی به نظر برسد:

```ini
# Production Master Secrets
APP_ENV=production
APP_KEY=base64:q9u4zX8YkP1vB5W2z+9Lm7T0aXp3Rt4Ys8U1e=
DB_CONNECTION=pgsql
DB_HOST=10.0.4.15
DB_PORT=5432
DB_DATABASE=app_production_db
DB_USERNAME=pg_master_admin
DB_PASSWORD=9k#Vp!82L_zTqX$921aA
AWS_ACCESS_KEY_ID=AKIAEXAMPLEHONEYPOT1
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
STRIPE_SECRET_KEY=sk_test_fake_key_for_honeypot_incident_response_demo
```

#### ۲. پایش در سطح کرنل با `auditd`:
استفاده از اسکریپت‌های پایتون یا `inotify` ممکن است با خاموش شدن پروسه متوقف شود، اما `auditd` مستقیماً در هسته لینوکس کار می‌کند:

```bash
# ایجاد رول مانیتورینگ برای رویدادهای Read، Write و Attribute Change
sudo auditctl -w /home/ubuntu/.env -p rwa -k honeypot_env_access
```

برای دائمی کردن این رول، آن را در انتهای فایل `/etc/audit/rules.d/audit.rules` ذخیره کنید:
```
-w /home/ubuntu/.env -p rwa -k honeypot_env_access
```

#### ۳. اسکریپت سرویس‌دهنده و قرنطینه خودکار شبکه (`Quarantine Daemon`):
یک سرویس پس‌زمینه با لاگ‌های `auditd` ارتباط برقرار کرده و به محض ثبت شدن رویداد با کلید `honeypot_env_access`، هاست را از شبکه ایزوله می‌کند:

```python
import subprocess
import time
import requests

AUDIT_LOG = "/var/log/audit/audit.log"
SLACK_WEBHOOK = "https://hooks.slack.com/services/XXX/YYY/ZZZ"

def quarantine_host(comm, pid):
    """قرنطینه کردن ماشین در سطح شبکه بدون قطع کردن کنسول اضطراری"""
    # بستن ترافیک شبکه داخلی برای جلوگیری از جابه‌جایی افقی (Lateral Movement)
    subprocess.run(["iptables", "-A", "OUTPUT", "-d", "10.0.0.0/8", "-j", "DROP"])
    subprocess.run(["iptables", "-A", "OUTPUT", "-d", "172.16.0.0/12", "-j", "DROP"])
    subprocess.run(["iptables", "-A", "OUTPUT", "-d", "192.168.0.0/16", "-j", "DROP"])
    
    # ارسال هشدار به تیم پاسخ به رخداد
    payload = {
        "text": f":fire: *CRITICAL SECURITY BREACH*: Honeypot `.env` accessed!\n"
                f"*Host:* `{subprocess.getoutput('hostname')}`\n"
                f"*Process:* `{comm}` (PID: `{pid}`)\n"
                f"*Action:* Network quarantined immediately. Memory dump recommended."
    }
    requests.post(SLACK_WEBHOOK, json=payload, timeout=3)

def tail_audit():
    p = subprocess.Popen(['tail', '-F', '-n', '0', AUDIT_LOG], stdout=subprocess.PIPE, text=True)
    for line in p.stdout:
        if "honeypot_env_access" in line and "SYSCALL" in line:
            # استخراج نام پروسس و شناسه پروسس
            parts = line.split()
            comm = [x.split('=')[1] for x in parts if x.startswith('comm=')][0].replace('"', '')
            pid = [x.split('=')[1] for x in parts if x.startswith('pid=')][0]
            quarantine_host(comm, pid)
            break

if __name__ == "__main__":
    tail_audit()
```

> **نکته جرم‌یابی:** با بستن ارتباط شبکه‌های داخلی، مهاجم ارتباطش با سایر سرورها قطع می‌شود اما به هیچ وجه ماشین را ریبوت نکنید؛ اجازه دهید کارشناس فارنزیک با استفاده از ابزارهایی مانند `LiME` یا `Volatility` حافظه RAM سرور را پیش از خاموشی تخلیه کند.

---

### هانی‌پات سوم: کلیدهای قناری (`Canarytokens`) در سکرت‌ها و ریپازیتوری‌ها

#### ۱. سازوکار توکن‌های AWS در `Canarytokens`:
پلتفرم متن‌باز **Canarytokens** با استفاده از زیرساخت اختصاصی خود، کلیدهای دسترسی استاندارد AWS (`AKIA...`) تولید می‌کند. این کلیدها در حساب‌های رسمی AWS ثبت شده‌اند، بنابراین وقتی مهاجم یا اسکنر اقدام به تست آن‌ها می‌کند، درخواست به سرورهای آمازون فرستاده شده و لاگ رخداد به سرعت در پنل ثبت می‌شود.

#### ۲. رفتار مهاجم به محض سرقت کلید:
هکر پس از استخراج کلید، ابتدا ابزار CLI آمازون را با این کلید پیکربندی کرده و یکی از متدهای زیر را اجرا می‌کند:

```bash
# رایج‌ترین دستور برای اعتبارسنجی اولیه
aws sts get-caller-identity

# یا تلاش برای لیست کردن باکت‌ها
aws s3 ls
```

به محض اجرای هر یک از این دستورات، توکن فعال شده و یک هشدار آنی شامل پارامترهای زیر ارسال می‌شود:
* تاریخ و زمان دقیق
* آدرس IP و موقعیت جغرافیایی درخواست‌کننده
* نوع درخواست API فراخوانی‌شده
* User-Agent ابزار مهاجم (مثلاً `aws-cli` یا اسکریپت پایتون سفارشی)

#### ۳. استراتژی برچسب‌گذاری و توزیع ساختاریافته (`Scoping & Tagging Strategy`):
اگر یک کلید را در ۵ سرور و ۳ مخزن گیت کپی کنید، هنگام به صدا درآمدن آلارم ساعت‌ها باید وقت بگذارید تا بفهمید کدام نقطه نشت کرده است.

**قاعده استاندارد نام‌گذاری توکن‌ها:**

```
canary-[محیط]-[نوع منبع]-[نام سرویس]-[مسیر یا کلید]
```

**نمونه‌های عملی:**
1. `canary-prod-repo-payment-service-ci-secrets` (قرار گرفته در سکرت‌های گیت‌هاب سرویس پرداخت)
2. `canary-prod-vm-edge-proxy-01-root-env` (قرار گرفته در فایل تنظیمات لبه)
3. `canary-stage-k8s-order-service-configmap` (قرار گرفته در ConfigMap سرویس سفارش‌ها)
4. `canary-corp-laptop-dev04-aws-credentials` (قرار گرفته در سیستم کارمند جهت تست امنیت داخلی)

#### ۴. گردش‌کار واکنش به نشت کلید (`Blast Radius Analysis`):
به محض دریافت ایمیل یا وب‌هوک از Canarytokens:
1. **تشخیص منبع:** نام توکن بلافاصله منبع نشت را مشخص می‌کند (مثلاً مخزن فلان در گیت‌هاب).
2. **بررسی دامنه نشت (Blast Radius):** تمامی کلیدها، پسوردها و توکن‌های واقعی که در همان مخزن یا در کنار آن کلید قناری قرار داشته‌اند، باید به عنوان «افشا شده» تلقی شوند و فوراً باطل (Revoke) گردند.
3. **بررسی دسترسی‌های اخیر به مخزن:** لاگ‌های دسترسی و کلون مخزن در گیت‌هاب/گیت‌لب بررسی شود تا اکانت هک‌شده توسعه‌دهنده کشف شود.

---

## ۳. اصول طلایی تله‌گذاری و راهنمای عملیاتی

1. **طبیعی‌سازی بافتاری (Contextual Plausibility):** اگر پشته فناوری شما کلاً بر پایه `Node.js` است، فایل `web.config` یا `appsettings.json` نگذارید! طعمه‌ها باید ۱۰۰٪ شبیه به فایل‌ها و متغیرهای واقعی سیستم‌های شما باشند.
2. **حفظ ایمنی دامی (No Real Secrets):** هرگز از بخشی از پسوردهای واقعی یا توکن‌های منقضی‌شده سازمان استفاده نکنید؛ هانی‌پات باید داده کاملاً تصادفی و تله‌ای باشد.
3. **دفترچه پاسخ به رخداد (Incident Runbook):** برای هر هانی‌پات باید مشخص باشد چه کسی مسئول دریافت هشدار است و اولین دستور برای قرنطینه یا ابطال دسترسی چیست تا در شرایط بحرانی زمان تلف نشود.
4. **مستندسازی محرمانه:** فهرست کامل محل قرارگیری تله‌ها را در ابزار مدیریت رمز عبور تیم امنیت ثبت کنید تا در آزمون‌های نفوذ دوره‌ای موجب اتلاف زمان کارشناسان خودی نشود.
