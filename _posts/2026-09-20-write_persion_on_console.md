---
title: "چاپ متن‌های فارسی در Console"
categories:
  - CSharp
tags:
  - csharp
  - console
  - net
---

در صورتی که نیاز دارید متن‌های فارسی را در کنسول نشان دهید و یا لاگی دارید که توسط Serilog در کنسول نوشته می‌شود بصورت پیش‌فرض به ازای هر حرف `?` نمایش داده می‌شود و با تنظیم `UTF-8` نیز حروف فارسی بصورت جدا و برعکس نشان داده می‌شوند.  
برای حل این مشکل می‌توانید از این کد استفاده کنید تا زبان فارسی به درستی نشان داده شود.  
کافی است این کد را برنامه خود قرار دهید.  

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Runtime.InteropServices;
using System.Text;

namespace Asa.RayanDataReceiver
{
    internal readonly struct GlyphForms
    {
        public readonly char Isolated;
        public readonly char Initial;
        public readonly char Medial;
        public readonly char Final;

        public GlyphForms(char isolated, char initial, char medial, char final)
        {
            Isolated = isolated;
            Initial = initial;
            Medial = medial;
            Final = final;
        }
    }

    /// <summary>
    /// موتور Shaping متن فارسی: تشخیص Runهای متوالی حروف فارسی در یک رشته (حتی چندخطی و
    /// مخلوط با لاتین/اعداد - مثل خطوط لاگ یا Stack Trace)، تبدیل هر حرف به شکل صحیح آن
    /// (Isolated/Initial/Medial/Final طبق Joining Type استاندارد یونیکد) و بازچینی
    /// راست‌به‌چپ فقط برای همان Run، بدون دست‌زدن به بخش‌های لاتین/عددی مجاور.
    /// این متد Idempotent است: صدا زدنش روی متنی که قبلاً Shape شده، تغییری ایجاد نمی‌کند
    /// (چون کاراکترهای حالت نهایی/میانی در جدول پایه نیستند)، پس اعمال دوباره‌اش بی‌خطر است.
    /// </summary>
    public static class PersianText
    {
        private static readonly Dictionary<char, GlyphForms> ShapeTable = new Dictionary<char, GlyphForms>
        {
            { '\u0627', new GlyphForms('\uFE8D', '\0', '\0', '\uFE8E') },
            { '\u0628', new GlyphForms('\uFE8F', '\uFE91', '\uFE92', '\uFE90') },
            { '\u067E', new GlyphForms('\uFB56', '\uFB58', '\uFB59', '\uFB57') },
            { '\u062A', new GlyphForms('\uFE95', '\uFE97', '\uFE98', '\uFE96') },
            { '\u062B', new GlyphForms('\uFE99', '\uFE9B', '\uFE9C', '\uFE9A') },
            { '\u062C', new GlyphForms('\uFE9D', '\uFE9F', '\uFEA0', '\uFE9E') },
            { '\u0686', new GlyphForms('\uFB7A', '\uFB7C', '\uFB7D', '\uFB7B') },
            { '\u062D', new GlyphForms('\uFEA1', '\uFEA3', '\uFEA4', '\uFEA2') },
            { '\u062E', new GlyphForms('\uFEA5', '\uFEA7', '\uFEA8', '\uFEA6') },
            { '\u062F', new GlyphForms('\uFEA9', '\0', '\0', '\uFEAA') },
            { '\u0630', new GlyphForms('\uFEAB', '\0', '\0', '\uFEAC') },
            { '\u0631', new GlyphForms('\uFEAD', '\0', '\0', '\uFEAE') },
            { '\u0632', new GlyphForms('\uFEAF', '\0', '\0', '\uFEB0') },
            { '\u0698', new GlyphForms('\uFB8A', '\0', '\0', '\uFB8B') },
            { '\u0633', new GlyphForms('\uFEB1', '\uFEB3', '\uFEB4', '\uFEB2') },
            { '\u0634', new GlyphForms('\uFEB5', '\uFEB7', '\uFEB8', '\uFEB6') },
            { '\u0635', new GlyphForms('\uFEB9', '\uFEBB', '\uFEBC', '\uFEBA') },
            { '\u0636', new GlyphForms('\uFEBD', '\uFEBF', '\uFEC0', '\uFEBE') },
            { '\u0637', new GlyphForms('\uFEC1', '\uFEC3', '\uFEC4', '\uFEC2') },
            { '\u0638', new GlyphForms('\uFEC5', '\uFEC7', '\uFEC8', '\uFEC6') },
            { '\u0639', new GlyphForms('\uFEC9', '\uFECB', '\uFECC', '\uFECA') },
            { '\u063A', new GlyphForms('\uFECD', '\uFECF', '\uFED0', '\uFECE') },
            { '\u0641', new GlyphForms('\uFED1', '\uFED3', '\uFED4', '\uFED2') },
            { '\u0642', new GlyphForms('\uFED5', '\uFED7', '\uFED8', '\uFED6') },
            { '\u06A9', new GlyphForms('\uFB8E', '\uFB90', '\uFB91', '\uFB8F') },
            { '\u06AF', new GlyphForms('\uFB92', '\uFB94', '\uFB95', '\uFB93') },
            { '\u0644', new GlyphForms('\uFEDD', '\uFEDF', '\uFEE0', '\uFEDE') },
            { '\u0645', new GlyphForms('\uFEE1', '\uFEE3', '\uFEE4', '\uFEE2') },
            { '\u0646', new GlyphForms('\uFEE5', '\uFEE7', '\uFEE8', '\uFEE6') },
            { '\u0648', new GlyphForms('\uFEED', '\0', '\0', '\uFEEE') },
            { '\u0647', new GlyphForms('\uFEE9', '\uFEEB', '\uFEEC', '\uFEEA') },
            { '\u06CC', new GlyphForms('\uFBFC', '\uFBFE', '\uFBFF', '\uFBFD') },
            { '\u0629', new GlyphForms('\uFE93', '\0', '\0', '\uFE94') },
            { '\u0622', new GlyphForms('\uFE81', '\0', '\0', '\uFE82') },
            { '\u0623', new GlyphForms('\uFE83', '\0', '\0', '\uFE84') },
            { '\u0624', new GlyphForms('\uFE85', '\0', '\0', '\uFE86') },
            { '\u0625', new GlyphForms('\uFE87', '\0', '\0', '\uFE88') },
            { '\u0626', new GlyphForms('\uFE89', '\uFE8B', '\uFE8C', '\uFE8A') },
            { '\u0621', new GlyphForms('\uFE80', '\0', '\0', '\0') },
        };

        private static readonly HashSet<char> DualJoining = new HashSet<char>
        {
            '\u0626', '\u0628', '\u062A', '\u062B', '\u062C', '\u062D', '\u062E', '\u0633',
            '\u0634', '\u0635', '\u0636', '\u0637', '\u0638', '\u0639', '\u063A', '\u0641',
            '\u0642', '\u0644', '\u0645', '\u0646', '\u0647', '\u067E', '\u0686', '\u06A9',
            '\u06AF', '\u06CC'
        };

        private static readonly HashSet<char> RightJoining = new HashSet<char>
        {
            '\u0622', '\u0623', '\u0624', '\u0625', '\u0627', '\u0629', '\u062F', '\u0630',
            '\u0631', '\u0632', '\u0648', '\u0698'
        };

        private static readonly HashSet<char> NonJoining = new HashSet<char> { '\u0621' };

        private static bool _initialized;
        private static bool _terminalSupportsShaping;

        /// <summary>
        /// باید فقط یک‌بار، در همان اولین خط Main، قبل از هر Console.Write یا ساخت
        /// LoggerConfiguration فراخوانی شود. Encoding کنسول را UTF-8 می‌کند، ترمینال میزبان
        /// را تشخیص می‌دهد، و Console.Out/Console.Error را با نسخه‌ی Shaping-Aware عوض می‌کند.
        /// </summary>
        public static void Initialize()
        {
            if (_initialized)
            {
                return;
            }

            // مرحله ۱: تنظیم Encoding - این کار Console.Out را داخلاً بازسازی می‌کند،
            // پس باید قبل از گرفتن رفرنس از Console.Out انجام شود.
            try
            {
                Console.OutputEncoding = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
                Console.InputEncoding = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
            }
            catch (IOException) { /* خروجی Redirect شده به فایل/پایپ؛ بی‌خطر */ }
            catch (PlatformNotSupportedException) { /* محیط محدود؛ بی‌خطر */ }

            _terminalSupportsShaping = DetectShapingCapableTerminal();

            // مرحله ۲: حالا که Console.Out نسخه‌ی نهایی (با Encoding درست) است، آن را می‌پیچیم.
            Console.SetOut(new PersianConsoleWriter(Console.Out));
            Console.SetError(new PersianConsoleWriter(Console.Error));

            _initialized = true;
        }

        /// <summary>
        /// متن ورودی را برای نمایش صحیح در کنسول اصلاح می‌کند. اگر ترمینال میزبان خودش
        /// Shaping/BiDi دارد (Windows Terminal, VS Code Terminal, اکثر ترمینال‌های
        /// لینوکس/مک)، متن دست‌نخورده برمی‌گردد.
        /// </summary>
        public static string Shape(string text)
        {
            if (!_initialized)
            {
                Initialize();
            }

            if (string.IsNullOrEmpty(text) || _terminalSupportsShaping)
            {
                return text;
            }

            var sb = new StringBuilder(text.Length);
            var i = 0;
            while (i < text.Length)
            {
                if (IsPersianLetter(text[i]))
                {
                    var start = i;
                    var end = i;
                    var lastLetterEnd = i;
                    while (end < text.Length && (IsPersianLetter(text[end]) || text[end] == ' '))
                    {
                        if (IsPersianLetter(text[end]))
                        {
                            lastLetterEnd = end + 1;
                        }

                        end++;
                    }
                    end = lastLetterEnd; // فاصله‌های انتهایی که به کلمه‌ی فارسی بعدی متصل نیستند حذف می‌شوند
                    sb.Append(ShapeAndReverseRun(text.Substring(start, end - start)));
                    i = end;
                }
                else
                {
                    sb.Append(text[i]);
                    i++;
                }
            }

            return sb.ToString();
        }

        private static bool IsPersianLetter(char c) => ShapeTable.ContainsKey(c);

        private static string ShapeAndReverseRun(string run)
        {
            var shaped = new char[run.Length];

            for (var k = 0; k < run.Length; k++)
            {
                var ch = run[k];

                if (!ShapeTable.TryGetValue(ch, out var forms))
                {
                    shaped[k] = ch; // فاصله‌ی داخلی بین دو کلمه‌ی فارسی
                    continue;
                }

                if (NonJoining.Contains(ch))
                {
                    shaped[k] = forms.Isolated;
                    continue;
                }

                var prevChar = k > 0 ? run[k - 1] : '\0';
                var nextChar = k < run.Length - 1 ? run[k + 1] : '\0';

                var joinsPrev = prevChar != '\0'
                                 && DualJoining.Contains(prevChar)
                                 && (DualJoining.Contains(ch) || RightJoining.Contains(ch));

                var joinsNext = DualJoining.Contains(ch)
                                 && nextChar != '\0'
                                 && (DualJoining.Contains(nextChar) || RightJoining.Contains(nextChar));

                char result;
                if (joinsPrev && joinsNext)
                {
                    result = forms.Medial != '\0' ? forms.Medial : forms.Final;
                }
                else if (joinsPrev)
                {
                    result = forms.Final;
                }
                else if (joinsNext)
                {
                    result = forms.Initial != '\0' ? forms.Initial : forms.Isolated;
                }
                else
                {
                    result = forms.Isolated;
                }

                shaped[k] = result;
            }

            Array.Reverse(shaped);
            return new string(shaped);
        }

        private static bool DetectShapingCapableTerminal()
        {
            if (!RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
            {
                return true; // لینوکس/مک: اکثر ترمینال‌های مدرن خودشان Shaping/BiDi دارند
            }

            var isWindowsTerminal = !string.IsNullOrEmpty(Environment.GetEnvironmentVariable("WT_SESSION"));
            var isVsCodeTerminal = Environment.GetEnvironmentVariable("TERM_PROGRAM") == "vscode";
            return isWindowsTerminal || isVsCodeTerminal;
        }
    }

    /// <summary>
    /// TextWriter‌ای که یک TextWriter دیگر (مثلاً Console.Out اصلی) را می‌پیچد و قبل از
    /// نوشتن، متن را از PersianText.Shape رد می‌کند. با override کردن فقط چند متد کلیدی،
    /// تمام Overloadهای Write/WriteLine (int, bool, object, char[], double, ...) به‌طور
    /// خودکار پوشش داده می‌شوند؛ چون پیاده‌سازی پایه‌ی TextWriter در .NET، آن Overloadها را
    /// با صدا زدن Write(string)/WriteLine(string) مجازی (virtual) پیاده‌سازی می‌کند.
    /// محدودیت شناخته‌شده: اگر کدی حرف‌به‌حرف با Write(char) بنویسد (نه رشته‌ی کامل)،
    /// Shaping روی هر حرف به‌تنهایی معنا ندارد و بدون تغییر عبور می‌کند - این الگو در
    /// Console.WriteLine، Console.Write(string)، ex.ToString() و خروجی سریلاگ دیده نمی‌شود.
    /// </summary>
    public sealed class PersianConsoleWriter : TextWriter
    {
        private readonly TextWriter _inner;

        public PersianConsoleWriter(TextWriter inner)
        {
            _inner = inner ?? throw new ArgumentNullException(nameof(inner));
        }

        public override Encoding Encoding => _inner.Encoding;

        public override string NewLine
        {
            get => _inner.NewLine;
            set => _inner.NewLine = value;
        }

        public override void Write(string value) => _inner.Write(PersianText.Shape(value));

        public override void Write(char[] buffer, int index, int count) =>
            _inner.Write(PersianText.Shape(new string(buffer, index, count)));

        public override void WriteLine() => _inner.WriteLine();

        public override void WriteLine(string value) => _inner.WriteLine(PersianText.Shape(value));

        public override void Write(char value) => _inner.Write(value);

        public override void Flush() => _inner.Flush();
    }
}
```

سپس در بخش شروع برنامه خود این خط را قرار دهید:  

```csharp
PersianText.Initialize();
```

اکنون بصورت خودکار جملات فارسی درست نشان داده می‌شوند. هرجا که از `Console.WriteLine` یا `Log.Error` یا `throw new Exception` استفاده شده باشد
