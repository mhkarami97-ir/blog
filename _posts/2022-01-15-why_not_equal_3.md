---
title: "چرا 0.1 + 0.2 برابر با 0.3 نیست؟!"
categories:
  - Trick
tags:
  - trick
  - math
  - binary
---

```c#
X = (sign) m * r^e
```

```c#
Console.WriteLine("{0:R}", .1 + .2); // 0.30000000000000004

Console.WriteLine("{0:R}", .1f + .2f); // 0.3

Console.WriteLine("{0:R}", .1m + .2m); // 0.3
```

```c#
float x = 0.1f; // 0.1 as a floating point
float y = 0.2f; // 0.2 as a floating point
float sum = x + y; // Calc the sum of both variables
bool same = sum == 0.3f; // Compare with fp of 0.3
Console.WriteLine(same); // True


double x = 0.1; // 0.1 as floating point double precision
double y = 0.2; // 0.2 as floating point double precision
double sum = x + y; // Calc the sum of both variables
bool same = sum == 0.3; // Compare with 0.3 floating point double precision
Console.WriteLine(same); // False
```

[wikipedia](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)  

[info](https://workitthrough.wordpress.com/2020/04/14/0-1-0-2-is-not-0-3/)  

[ieeefloat](https://steve.hollasch.net/cgindex/coding/ieeefloat.html)  

[0.30000000000000004](https://0.30000000000000004.com/)  

[bug](https://ctftime.org/writeup/9913)  