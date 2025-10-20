---
title: "استفاده از Vault بجای AppSetting در .Net"
categories:
  - Net
tags:
  - net
  - vault
  - appsetting
---

Vault یک سیستم مدیریت کلید (Secrets Management) است که برای ذخیره‌سازی، کنترل دسترسی و مدیریت اطلاعات حساس مانند رمزهای عبور، توکن‌های API، کلیدهای رمزنگاری و گواهینامه‌های دیجیتال طراحی شده است. این ابزار توسط HashiCorp توسعه یافته و به سازمان‌ها کمک می‌کند تا امنیت داده‌های محرمانه خود را در محیط‌های ابری، داخلی و ترکیبی تضمین کنند.  

ویژگی‌های کلیدی Vault  
 - ذخیره‌سازی امن: اطلاعات حساس را در یک فضای رمزگذاری‌شده ذخیره می‌کند و دسترسی به آن‌ها را محدود می‌نماید.  
 - رمزگذاری به‌عنوان سرویس (Encryption as a Service): امکان رمزگذاری و رمزگشایی داده‌ها بدون نیاز به افشای کلیدهای رمزنگاری.  
 - احراز هویت پویا (Dynamic Secrets): تولید و اعتبارسنجی خودکار اطلاعات حساس، مانند دسترسی موقتی به پایگاه داده‌ها یا سرویس‌های ابری.  
 - کنترل دسترسی دقیق (Policy-Based Access Control): تعریف سطوح دسترسی برای کاربران و سرویس‌ها بر اساس سیاست‌های مشخص.  
 - چرخش خودکار رمزها (Automatic Secret Rotation): امکان تغییر و بازنشانی خودکار رمزهای عبور و کلیدها برای کاهش خطرات امنیتی.  
 - مدیریت چند ابری (Multi-Cloud Integration): پشتیبانی از AWS, Azure, Google Cloud و دیگر پلتفرم‌های ابری.  

### روش استفاده

ابتدا پکیج زیر را به پروژه خود اضافه کنید:  

[VaultSharp.Extensions.Configuration](https://www.nuget.org/packages/VaultSharp.Extensions.Configuration)  

بعد از نصب پکیج کافی است کلاس زیر را به سیستم خود اضافه کنید تا تنظیمات مربوط به Vault انجام شود.  

```csharp
public static class Vault
{
    public static void ConfigureVault(this WebApplicationBuilder builder)
    {
        if (builder.Environment.IsDevelopment()) return;

        using var loggerFactory = LoggerFactory.Create(config => config.AddConsole().AddDebug());
        var logger = loggerFactory.CreateLogger<Program>();

        var vaultConfig = builder.Configuration.GetSection(nameof(VaultConfig)).Get<VaultConfig>() ??
                          throw new LogicException($"{nameof(VaultConfig)} is null", string.Empty, ExceptionLevel.Error, null);

        builder.Configuration
            .AddEnvironmentVariables()
            .AddVaultConfiguration(
                () => new VaultOptions(
                    vaultConfig.VaultAddress,
                    authMethod: new UserPassAuthMethodInfo(vaultConfig.UserPassAuthInfo.UserName, vaultConfig.UserPassAuthInfo.Password),
                    reloadOnChange: vaultConfig.ReloadOnChange,
                    reloadCheckIntervalSeconds: vaultConfig.ReloadCheckIntervalSeconds,
                    insecureConnection: vaultConfig.AcceptInsecureConnection),
                vaultConfig.BasePath,
                vaultConfig.MountPoint,
                logger);

        if (vaultConfig.ReloadOnChange)
            builder.Services.AddHostedService<VaultChangeWatcher>();
    }
}
```

اکنون در `Program.cs` خط زیر را اضافه کنید.  

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.ConfigureVault();
```

اکنون `AppSetting.json` خود را ویرایش کنید و مقادیر زیر را در آن قرار دهید.  
دقت کنید که مقادیر موجود در `AppSetting.Development.json` همان مقادیر قبلی شما هستند تا بتوانید در سیستم Local بدون نیاز به Vault با سیستم کار کنید.  

```csharp
{
  "VaultConfig": {
    "VaultAddress": "https://vault.my.local:8100",
    "BasePath": "Project/pr",
    "MountPoint": "local",
    "ReloadOnChange": false,
    "ReloadCheckIntervalSeconds": 300,
    "AcceptInsecureConnection": true,
    "UserPassAuthInfo": {
      "UserName": "local-viewer",
      "Password": "MyPass!@#"
    }
  }
}

```

اکنون کافی است Json کلیدهای قبلی موجود در AppSetting.json را در UI موجود برای Vault قرار دهید تا بتوانید در سیستم خود از آن استفاده کنید.  

![mhkarami97](/assets/img/vault.jpg)  