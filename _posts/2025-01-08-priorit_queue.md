---
title: "صف دارای اولویت یا PriorityQueue"
categories:
  - Net
tags:
  - priority_queue
  - net
  - queue
---

توسط نوع داده PriorityQueue می‌توانید اولویت را هم در یک لیست دخیل کنید.  
با این روش هر دیتا یک اولویت هم دارد که در زمان خواندن دیتا این اولویت هم بررسی می‌شود و دیتایی که اولویت کمتری دارد سریعتر برداشته می‌شود.  
همچنین در زمان اضافه کردن دیتا نیز ترتیب دیتا عوض می‌شود و بر اساس اولویت مرتب می‌شود.  

[PriorityQueue](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.priorityqueue-2?view=net-9.0)  

نمونه استفاده:  

```csharp
var queue = new PriorityQueue<string, int>();

queue.Enqueue("Task 1", 3);
queue.Enqueue("Task 2", 1);
queue.Enqueue("Task 3", 2);

// Dequeue the elements in order of priority (highest priority first)
while (queue.Count > 0)
{
    if (queue.TryDequeue(out var task, out var priority))
    {
        Console.WriteLine($"Dequeued: {task} with priority {priority}");
    }
}
```

زمان اضافه کردن و دریافت دیتا O(log n) است.  
در پشت این نوع داده از Binary Heap استفاده شده است.  
همچنین این نوع داده بصورت Thread Safe نیست.  