---
title: "تغییر فراخوانی API از Client Side در مرورگر به Server Side در انگولار"
categories:
  - Angular
tags:
  - angular
  - api_call
  - proxy
  - nginx
---

فرض کنید پروژه Angular خود را در سروری با اسم st-server-s2 هاست کرده‌اید و در داخل این پروژه چند api از سرور دیگری با اسم st-backend-s2 فراخوانی می‌کنید. دسترسی شبکه‌ای بین 2 سرور گفته شده برقرار است و می‌توانید از سرور UI به سرور دوم Telnet بگیرید. با توجه به اینکه API Call در انگولار بطور پیش‌فرض ClientSide است و در مرورگر کاربر به سرور دوم ریکوست زده می‌شود، برای کارکرد درست باید دسترسی شبکه‌ای کاربر را هم به st-server-s2 و هم st-backend-s2 بگیرید در غیر این صورت فقط UI را می‌توانید مشاهده کنید و فراخوانی سرویس‌ها کار نمی‌کند.  
برای برطرف کردن این مشکل و همچنین بدون نیاز به گرفتن دسترسی به سرور دوم می‌توان از Proxy در انگولار استفاده کرد. با این روش فراخوانی API از سرور st-server-s2 انجام می‌شود و دیگر نیاز به دسترسی بیشتر نیست.  

بدین منظور ابتدا در پوشه src فایلی به اسم `proxy.conf.json` ایجاد کنید و مقادیر آن را بصورت زیر تعیین کنید:  

```csharp
{
  "/support-api/**": {
    "target": "http://st-backend-s2:50609",
    "secure": false,
    "changeOrigin": true,
    "logLevel": "debug",
    "pathRewrite": {
      "^/support-api": ""
    }
  }
}
```

همچنین اگر نیاز به تغییر آن در محیط‌های مختلف مانند Stage, Prod دارید می‌توانید فایل دیگر به اسم `proxy.prod.conf.json` ایجاد کنید:  

```csharp
{
  "/support-api/**": {
    "target": "http://st-prod-backend-s2:50609",
    "secure": false,
    "changeOrigin": true,
    "logLevel": "debug",
    "pathRewrite": {
      "^/support-api": ""
    }
  }
}
```

در کد بالا یک آدرس به اسم `support-api` تعریف شده است، سپس در target آدرس نهایی که api در آن قرار دارد گذاشته شده است.  
در بخش pathRewrite نیز گفته شده است که ابتدا support-api قبل از فراخوانی target پاک شود و سپس فراخوانی شود.  
بطور مثال آدرس زیر را در نظر بگیرید:  

`http://st-server-s2/support-api/api/v1/customer/get`
این آدرس توسط این فایل به این تغییر خواهد کرد:  
`http://st-backend-s2:50609/api/v1/customer/get`
  
  

در ادامه برای اینکه خود support-api را در محیط‌های مختلف تعریف کنید می‌توانید از فایل `environment.ts` استفاده کنید.  
شبیه به کد بالا می‌توانید فایل `environment.prod.ts` را نیز ایجاد کنید.  

```csharp
export const environment = {
  production: false,
  baseUrl: '/support-api'
};
```
در ادامه در فایل `Angular.json` نیاز است تغییرات زیر را اعمال کنید:  
در بخش `serve => options` مقدار `"proxyConfig": "src/proxy.conf.json"` را اضافه کنید.  
در بخش `build => configurations` به ازای محیط خود یک بخش جدید مانند staging ایجاد کنید و در بخش fileReplacements آن بصورت زیر عمل کنید.  

```csharp
"architect": {
        "build": {
          "configurations": {
            "staging": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.staging.ts"
                },
                {
                  "replace": "src/proxy.conf.json",
                  "with": "src/proxy.staging.conf.json"
                }
              ]
            },
            "beta": {          
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.beta.ts"
                },
                {
                  "replace": "src/proxy.conf.json",
                  "with": "src/proxy.beta.conf.json"
                }
              ]
            }
          },
        },
        "serve": {
          "options": {
            "proxyConfig": "src/proxy.conf.json"
          }
        },
      }
```
در فایل `package.json` نیز می‌توانید build با کانفیگ گفته شده را ایجاد کنید:  

```typescript
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "build:prod": "npm run build -- --configuration production --aot",
    "build:staging": "npm run build -- --configuration staging --aot",
  },
```

نکته مهم این است که برای اینکه تمام موارد بالا بعد از پابلیش در nginx یا iis به درستی کار کند نیاز به تغییر کانفیگ nginx است.  
ابتدا یک فایل به اسم nginx.conf در روت پروژه خود ایجاد کنید.  

مشکل اول routing در انگولار است که اگر بصورت مثال آدرس `st-server-s2/page/customer` را در یک تب جدید مرورگر کپی کنید خطا 404 می‌گیرید. برای حل آن کافی است بخش اول کد زیر که `location / ` است را در کد خود قرار بدهید.  
برای اینکه proxy نیز به درستی کار کند نیاز است بخش `location /support-api` را نیز در کانفیگ خود قرار دهید. در بخش proxy-pass نیز آدرس api خود را بگذارید.  
برای اینکه در محیط‌های مختلف نیز بتوانید آدرس را تغییر دهید کافی است فایل جدید به اسم nginx.prod.conf ایجاد کنید. البته این فایل را مانند روش‌های گفته شده در بالا نمی‌توان در Angular.json قرار داد و نیاز است در PipeLine پابلیش خود در هر محیط فایل مخصوص آن را معرفی کنید.  

```csharp
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

     location /support-api {
        rewrite ^/support-api/(.*)$ /$1 break;
        proxy_pass http://st-backend-s2:50609;
     }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

در بخش Service خود کافی است مقدار basePath را به مقداری که در proxy تعیین کرده بودید تغییر بدهید تا از این به بعد فراخوانی سرویس‌ها به آن آدرس ارسال شود.  

```typescript
@Injectable()
export class InfoService {

  protected basePath = '/support-api';
  public defaultHeaders = new HttpHeaders();
  public configuration = new Configuration();

  constructor(protected httpClient: HttpClient, @Optional() @Inject(BASE_PATH) basePath: string, @Optional() configuration: Configuration) {
    if (basePath) {
      this.basePath = basePath;
    }
    if (configuration) {
      this.configuration = configuration;
      this.basePath = basePath || configuration.basePath || this.basePath;
    }
  }
}
```

برای اینکه مقدار basePath در محیط‌های مختلف از فایل Environment خوانده شود هم کافی است در AppModule خود بصورت زیر BASE_PATH را تعریف کنید.  

```typescript
@NgModule({
  imports: [
    ApiModule.forRoot(() => new Configuration()),
  ],
  providers: [
    {provide: BASE_PATH, useValue: environment.baseUrl},
  ],
})
export class AppModule {
}
```
اکنون اگر پروژه خود را ران کنید به درستی کار می‌کند.  