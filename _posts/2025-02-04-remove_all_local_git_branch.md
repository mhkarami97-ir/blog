---
title: "پاک کردن تمام برنچ‌های Local در Git"
categories:
  - Git
tags:
  - git
  - branch
  - local
---

در صورتی که نیاز داشتید تا تمام Branch های گیت در سیستم لوکال خود را بجز یک برنچ خاص بطور مثال Master پاک کنید می‌توانید از دستور زیر استفاده کنید.  
برای این کار ابتدا در پوشه پروژه خود راست کلیک کنید و گزینه `Open Git Bash Here` را انتخاب کنید.  

![mhkarami97](/assets/img/git_branch.jpg)  

سپس کافی است دستور زیر را وارد کنید تا برنچ‌ها پاک شوند.  

```
git branch | grep -v "master" | xargs git branch -D
```

![mhkarami97](/assets/img/git_branch1.jpg)  