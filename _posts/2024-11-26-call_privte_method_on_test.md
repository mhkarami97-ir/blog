---
title: "فراخوانی متودهای Private یک کلاس در تست‌ها"
categories:
  - Net
tags:
  - net
  - private
  - test
---

در مواقعی که در تست‌های خود نیاز داشتید تا متودهای پرایوت یک کلاس را فراخوانی کنید، می‌توانید از تکه کد زیر استفاده کنید.  


```csharp
var myClass= new MyClass();
var myData= 10;

var myMethod = myClass.GetType().GetMethod("MyMethod", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);
myMethod?.Invoke(myClass, [myData]);
```

```csharp
public class MyClass
{
    private void MyMethod(int myData){
    }
}
```
