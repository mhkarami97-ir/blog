---
title: "مسدود سازی دسترسی یک برنامه به اینترنت"
categories:
  - Trick
tags:
  - internet
  - block
  - windows
---

در صورتی که می‌خواهید دسترسی یک برنامه را بصورت کامل به اینترنت ببندید می‌توانید از کوئری زیر استفاده کنید.  
با این روش بصورت کامل تمام دسترسی‌های برنامه خواسته شده به اینترنت بسته می‌شود.  
دقت کنید که در کوئری زیر نیاز است مقدار `appPath` را به آدرس برنامه خود تغییر دهید.  

## روش انجام
ابتدا یک فایل txt بسازید و کد زیر را در آن قرار دهید.  
سپس آن را با نام دلخواه ذخیره کنید.  
سپس کافی است فرمت فایل را از `txt` به `ps1` تغییر دهید. مانند `blocker.ps1`

```powershell
#Requires -RunAsAdministrator
$appPath = "C:\Program Files (x86)\Poedit\Poedit.exe"
$ruleBaseName = "Block Poedit"
$profiles = @("Domain", "Private", "Public")
$protocols = @("TCP", "UDP")
$directions = @("Outbound", "Inbound")

if (-not (Test-Path $appPath)) {
    Write-Error "Path not found: $appPath. Verify Poedit installation path before running this script."
    exit 1
}

foreach ($direction in $directions) {
    foreach ($protocol in $protocols) {
        $ruleName = "$ruleBaseName - $direction - $protocol"

        $existingRule = Get-NetFirewallRule -DisplayName $ruleName -ErrorAction SilentlyContinue
        if ($existingRule) {
            Write-Host "Rule already exists, removing old rule: $ruleName" -ForegroundColor Yellow
            Remove-NetFirewallRule -DisplayName $ruleName
        }

        New-NetFirewallRule `
            -DisplayName $ruleName `
            -Direction $direction `
            -Program $appPath `
            -Protocol $protocol `
            -Action Block `
            -Profile ($profiles -join ",") `
            -Enabled True | Out-Null

        Write-Host "Created rule: $ruleName" -ForegroundColor Green
    }
}

Write-Host "`nAll firewall rules for Poedit.exe have been created successfully." -ForegroundColor Cyan
Write-Host "Verify with: Get-NetFirewallRule -DisplayName 'Block Poedit*' | Format-Table DisplayName,Direction,Action,Enabled"
```

اکنون کافی است `PowerShell` را در محلی که فایل قرار دارد باز کنید و سپس دستور زیر را بزنید:  
دقت کنید که وجود `.\` در دستور زیر مهم است.  

```powershell
.\blocker.ps1
```

در صورتی که با خطا مواجه شدید کافی است یکبار دستور زیر را بزنید و سپس دستور بالا را اجرا کنید:  

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

## نکات
- اگر مسیر نصب برنامه پس از آپدیت تغییر کند یا فایل exe جایگزین شود، Rule دیگر اثری نخواهد داشت و باید دوباره تعریف شود.
- برخی برنامه‌ها فرآیندهای کمکی جدا دارند (مثلاً یک helper.exe یا svchost که به‌جای آن ارتباط شبکه را برقرار می‌کند)؛ در صورتی که ارتباط همچنان برقرار باشد، این فرآیندهای کمکی نیز باید بررسی شوند.
- فایروال سطح کاربر (چه ویندوز، چه آنتی‌ویروس) در سطح سیستم‌عامل عمل می‌کند؛ برای ایزوله‌سازی واقعاً محکم (مثلاً برای فایلی که اعتماد کافی به آن وجود ندارد)، استفاده از یک ماشین مجازی بدون کارت شبکه واقعی یا اجرا در sandbox گزینه مطمئن‌تری است.