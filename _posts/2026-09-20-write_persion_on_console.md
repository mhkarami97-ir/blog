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
using System.IO;
using System.Runtime.InteropServices;
using System.Text;

namespace Asa.RayanDataReceiver
{
    /// <summary>
    /// یک Decorator بهینه‌شده برای TextWriter که محدودیت‌های کنسول ویندوز در نمایش 
    /// حروف به هم چسبیده و راست‌به‌چپ (RTL) زبان‌های فارسی و عربی را برطرف می‌کند.
    /// مجهز به سیستم تشخیص هوشمند برای جلوگیری از تداخل با ترمینال‌های مدرن.
    /// </summary>
    public sealed class PersianConsoleWriter : TextWriter
    {
        private readonly TextWriter _originalWriter;
        private readonly bool _isModernTerminal;

        // محدوده کاراکترهای عربی/فارسی در جدول یونیکد برای فیلترینگ بسیار سریع (O(N))
        private const char ArabicBlockStart = '\u0600';
        private const char ArabicBlockEnd = '\u06FF';

        public override Encoding Encoding => Encoding.UTF8;

        public PersianConsoleWriter(TextWriter originalWriter)
        {
            _isModernTerminal = CheckIfModernTerminal();
            SetEncoding(_isModernTerminal);
            _originalWriter = originalWriter ?? throw new ArgumentNullException(nameof(originalWriter));
        }

        public static void SetEncoding(bool isModernTerminal = false)
        {
            Console.OutputEncoding = Encoding.UTF8;
            Console.InputEncoding = Encoding.UTF8;

            // در ترمینال‌های مدرن نیازی به دستکاری فونت ویندوز نیست
            if (!isModernTerminal)
            {
                ConsoleFontHelper.SetConsolasFont();
            }
        }

        /// <summary>
        /// تشخیص خودکار محیط اجرا برای جلوگیری از دوبار معکوس شدن متن در ترمینال‌های هوشمند
        /// </summary>
        private static bool CheckIfModernTerminal()
        {
            // Windows Terminal
            if (!string.IsNullOrEmpty(Environment.GetEnvironmentVariable("WT_SESSION")))
            {
                return true;
            }

            // JetBrains Rider / IntelliJ
            if (!string.IsNullOrEmpty(Environment.GetEnvironmentVariable("TERMINAL_EMULATOR")))
            {
                return true;
            }

            // VS Code
            if (Environment.GetEnvironmentVariable("TERM_PROGRAM") == "vscode")
            {
                return true;
            }

            return false;
        }

        // 1. رهگیری WriteLine
        public override void WriteLine(string value)
        {
            if (string.IsNullOrEmpty(value))
            {
                _originalWriter.WriteLine(value);
                return;
            }

            if (_isModernTerminal || !ContainsPersian(value))
            {
                _originalWriter.WriteLine(value);
            }
            else
            {
                _originalWriter.WriteLine(PersianTextShaper.Process(value));
            }
        }

        // 2. رهگیری Write (نقطه فرار Serilog در اینجا بسته می‌شود)
        public override void Write(string value)
        {
            if (string.IsNullOrEmpty(value))
            {
                _originalWriter.Write(value);
                return;
            }

            if (_isModernTerminal || !ContainsPersian(value))
            {
                _originalWriter.Write(value);
            }
            else
            {
                _originalWriter.Write(PersianTextShaper.Process(value));
            }
        }

        // 3. رهگیری آرایه‌های کاراکتری (مسیر دوم فرار لاگرها)
        public override void Write(char[] buffer, int index, int count)
        {
            var text = new string(buffer, index, count);
            Write(text);
        }

        public override void Write(char[] buffer)
        {
            if (buffer == null)
            {
                return;
            }

            Write(new string(buffer));
        }

        public override void Write(char value)
        {
            _originalWriter.Write(value);
        }

        /// <summary>
        /// بررسی وجود حروف فارسی با استفاده از حلقه for ساده به جای LINQ 
        /// جهت جلوگیری از تخصیص حافظه (Zero Allocation) و حفظ حداکثر سرعت.
        /// </summary>
        private bool ContainsPersian(string text)
        {
            for (var i = 0; i < text.Length; i++)
            {
                var character = text[i];
                if (character >= ArabicBlockStart && character <= ArabicBlockEnd)
                {
                    return true;
                }
            }

            return false;
        }
    }

    /// <summary>
    /// هسته پردازشگر متن (Text Shaper). 
    /// وظیفه تبدیل کدهای استاندارد یونیکد به کاراکترهای چسبیده (Presentation Forms) 
    /// و مدیریت جهت متن (Bi-directional) را بر عهده دارد.
    /// </summary>
    internal static class PersianTextShaper
    {
        private const int LaaGlyphIndex = 400; // 8 * 50 (آفست برای ترکیبات حرف 'لا')
        private const string LeftConnectingChars = "یٹہےگکڤچپـئظشسيبلتنمكطضصثقفغعهخحج";
        private const string RightConnectingChars = "یٹہےڈڑگکڤژچپـئؤرلالآىآةوزظشسيبللأاأتنمكطضصثقفغعهخحجدذلإإۇۆۈ";
        private const string Symbols = @"ـ.،؟ @#$%^&*-+|\/=~,:";
        private const string Brackets = "(){}[]";
        private const string BaseArabicChars = "آأإابتثجحخدذرزسشصضطظعغفقكلمنهويةؤئىپچژڤگٹہےیڈڑۇۆۈک";
        private const string NonEnglishTerminators = BaseArabicChars + "ء،؟";

        // جدول نگاشت (Mapping Table) برای ۴ حالت ممکن هر حرف (تنها، آخر، اول، وسط)
        private const string PresentationForms =
            "ﺁ ﺁ ﺂ ﺂ " + "ﺃ ﺃ ﺄ ﺄ " + "ﺇ ﺇ ﺈ ﺈ " + "ﺍ ﺍ ﺎ ﺎ " + "ﺏ ﺑ ﺒ ﺐ " + "ﺕ ﺗ ﺘ ﺖ " +
            "ﺙ ﺛ ﺜ ﺚ " + "ﺝ ﺟ ﺠ ﺞ " + "ﺡ ﺣ ﺤ ﺢ " + "ﺥ ﺧ ﺨ ﺦ " + "ﺩ ﺩ ﺪ ﺪ " + "ﺫ ﺫ ﺬ ﺬ " +
            "ﺭ ﺭ ﺮ ﺮ " + "ﺯ ﺯ ﺰ ﺰ " + "ﺱ ﺳ ﺴ ﺲ " + "ﺵ ﺷ ﺸ ﺶ " + "ﺹ ﺻ ﺼ ﺺ " + "ﺽ ﺿ ﻀ ﺾ " +
            "ﻁ ﻃ ﻄ ﻂ " + "ﻅ ﻇ ﻈ ﻆ " + "ﻉ ﻋ ﻌ ﻊ " + "ﻍ ﻏ ﻐ ﻎ " + "ﻑ ﻓ ﻔ ﻒ " + "ﻕ ﻗ ﻘ ﻖ " +
            "ﻙ ﻛ ﻜ ﻚ " + "ﻝ ﻟ ﻠ ﻞ " + "ﻡ ﻣ ﻤ ﻢ " + "ﻥ ﻧ ﻨ ﻦ " + "ﻩ ﻫ ﻬ ﻪ " + "ﻭ ﻭ ﻮ ﻮ " +
            "ﻱ ﻳ ﻴ ﻲ " + "ﺓ ﺓ ﺔ ﺔ " + "ﺅ ﺅ ﺆ ﺆ " + "ﺉ ﺋ ﺌ ﺊ " + "ﻯ ﻯ ﻰ ﻰ " + "ﭖ ﭘ ﭙ ﭗ " +
            "ﭺ ﭼ ﭽ ﭻ " + "ﮊ ﮊ ﮋ ﮋ " + "ﭪ ﭬ ﭭ ﭫ " + "ﮒ ﮔ ﮕ ﮓ " + "ﭦ ﭨ ﭩ ﭧ " + "ﮦ ﮨ ﮩ ﮧ " +
            "ﮮ ﮰ ﮱ ﮯ " + "ﯼ ﯾ ﯿ ﯽ " + "ﮈ ﮈ ﮉ ﮉ " + "ﮌ ﮌ ﮍ ﮍ " + "ﯗ ﯗ ﯘ ﯘ " + "ﯙ ﯙ ﯚ ﯚ " +
            "ﯛ ﯛ ﯜ ﯜ " + "ﮎ ﮐ ﮑ ﮏ " + "ﻵ ﻵ ﻶ ﻶ " + "ﻷ ﻷ ﻸ ﻸ " + "ﻹ ﻹ ﻺ ﻺ " + "ﻻ ﻻ ﻼ ﻼ ";

        // ثابت‌های تعیین موقعیت حرف در کلمه
        private const int ShapeIsolated = 0;
        private const int ShapeFinal = 2;
        private const int ShapeInitial = 4;
        private const int ShapeMedial = 6;

        public static string Process(string input)
        {
            if (string.IsNullOrEmpty(input))
            {
                return input;
            }

            var chars = input.ToCharArray();
            var inputLength = chars.Length;

            // تخصیص یک‌باره حافظه (Pre-allocation) به جای استفاده از عملگر += 
            var buffer = new char[inputLength * 2];
            var bufferIndex = buffer.Length - 1;

            void PushToBuffer(char c)
            {
                buffer[bufferIndex--] = c;
            }

            for (var currentIndex = 0; currentIndex < inputLength; currentIndex++)
            {
                var shapePosition = ShapeIsolated;
                var currentCharacter = chars[currentIndex];

                // فاز ۱: تشخیص موقعیت کاراکتر در کلمه (Contextual Analysis)
                if (currentIndex == 0)
                {
                    shapePosition = RightConnectingChars.IndexOf(chars[0]) >= 0 ? ShapeFinal : ShapeIsolated;
                }
                else if (currentIndex == inputLength - 1)
                {
                    shapePosition = (inputLength > 1 && LeftConnectingChars.IndexOf(chars[inputLength - 2]) >= 0) ? ShapeMedial : ShapeIsolated;
                }
                else
                {
                    var isPrevLeftConnecting = LeftConnectingChars.IndexOf(chars[currentIndex - 1]) >= 0;
                    var isNextRightConnecting = RightConnectingChars.IndexOf(chars[currentIndex + 1]) >= 0;

                    if (!isPrevLeftConnecting)
                    {
                        shapePosition = !isNextRightConnecting ? ShapeIsolated : ShapeFinal;
                    }
                    else
                    {
                        shapePosition = isNextRightConnecting ? ShapeInitial : ShapeMedial;
                    }
                }

                // فاز ۲: نگاشت و پردازش کاراکترها
                if (currentCharacter == 'ء')
                {
                    PushToBuffer('ﺀ');
                }
                else if (Brackets.IndexOf(currentCharacter) >= 0)
                {
                    var bracketMatchIndex = Brackets.IndexOf(currentCharacter);
                    PushToBuffer(bracketMatchIndex % 2 == 0 ? Brackets[bracketMatchIndex + 1] : Brackets[bracketMatchIndex - 1]);
                }
                else if (BaseArabicChars.IndexOf(currentCharacter) >= 0)
                {
                    if (currentCharacter == 'ل' && currentIndex + 1 < inputLength)
                    {
                        var nextCharBaseIndex = BaseArabicChars.IndexOf(chars[currentIndex + 1]);
                        if (nextCharBaseIndex >= 0 && nextCharBaseIndex < 4) // مدیریت لیگچر (Ligature) 'لا'
                        {
                            PushToBuffer(PresentationForms[nextCharBaseIndex * 8 + shapePosition + LaaGlyphIndex]);
                            currentIndex++;
                        }
                        else
                        {
                            PushToBuffer(PresentationForms[BaseArabicChars.IndexOf(currentCharacter) * 8 + shapePosition]);
                        }
                    }
                    else
                    {
                        PushToBuffer(PresentationForms[BaseArabicChars.IndexOf(currentCharacter) * 8 + shapePosition]);
                    }
                }
                else if (Symbols.IndexOf(currentCharacter) >= 0)
                {
                    PushToBuffer(currentCharacter);
                }
                else if (PresentationForms.IndexOf(currentCharacter) >= 0)
                {
                    var unicodeIndex = PresentationForms.IndexOf(currentCharacter);
                    if (unicodeIndex >= LaaGlyphIndex)
                    {
                        for (var offset = 8; offset < 40; offset += 8)
                        {
                            if (unicodeIndex < offset + LaaGlyphIndex)
                            {
                                PushToBuffer(BaseArabicChars[(offset / 8) - 1]);
                                PushToBuffer('ل');
                                break;
                            }
                        }
                    }
                    else
                    {
                        for (var offset = 8; offset < LaaGlyphIndex + 32; offset += 8)
                        {
                            if (unicodeIndex < offset)
                            {
                                PushToBuffer(BaseArabicChars[(offset / 8) - 1]);
                                break;
                            }
                        }
                    }
                }
                else
                {
                    // فاز ۳: مدیریت کلمات انگلیسی و اعداد درون متن فارسی (Bi-directional Text Handling)
                    var segmentBuilder = new StringBuilder();
                    var lookAheadIndex = currentIndex;

                    while (inputLength > lookAheadIndex &&
                           NonEnglishTerminators.IndexOf(chars[lookAheadIndex]) < 0 &&
                           PresentationForms.IndexOf(chars[lookAheadIndex]) < 0 &&
                           Brackets.IndexOf(chars[lookAheadIndex]) < 0)
                    {
                        segmentBuilder.Append(NormalizeToPersianNumber(chars[lookAheadIndex]));
                        lookAheadIndex++;
                    }

                    var englishSegment = segmentBuilder.ToString();
                    var segmentEndIndex = englishSegment.Length - 1;
                    var trailingSpacesCount = 0;

                    // انتقال فاصله‌های انتهای بلوک انگلیسی به پشت بلوک برای رندر صحیح در چیدمان RTL
                    while (segmentEndIndex >= 0 && englishSegment[segmentEndIndex] == ' ')
                    {
                        trailingSpacesCount++;
                        segmentEndIndex--;
                    }

                    for (var i = segmentEndIndex; i >= 0; i--)
                    {
                        PushToBuffer(englishSegment[i]);
                    }

                    for (var i = 0; i < trailingSpacesCount; i++)
                    {
                        PushToBuffer(' ');
                    }

                    currentIndex = lookAheadIndex - 1;
                }
            }

            var finalStringLength = buffer.Length - bufferIndex - 1;
            return new string(buffer, bufferIndex + 1, finalStringLength);
        }

        /// <summary>
        /// تبدیل ریاضیاتی (O(1)) اعداد انگلیسی یا عربی به اعداد فارسی.
        /// </summary>
        private static char NormalizeToPersianNumber(char character)
        {
            if (character >= '0' && character <= '9')
            {
                return (char)(character - '0' + '۰');
            }

            if (character >= '٠' && character <= '٩')
            {
                return (char)(character - '٠' + '۰');
            }

            return character;
        }
    }

    /// <summary>
    /// ابزاری برای تعامل با APIهای سطح پایین ویندوز جهت تغییر فونت کنسول.
    /// به صورت امن پیاده‌سازی شده تا در صورت عدم دسترسی، باعث توقف برنامه نشود.
    /// </summary>
    public static class ConsoleFontHelper
    {
        private const int STD_OUTPUT_HANDLE = -11;
        private const int LF_FACESIZE = 32;

        [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
        private struct CONSOLE_FONT_INFO_EX
        {
            public uint cbSize;
            public uint nFont;
            public COORD dwFontSize;
            public int FontFamily;
            public int FontWeight;

            // استفاده از MarshalAs به جای fixed char برای جلوگیری از نیاز به کامپایل unsafe
            [MarshalAs(UnmanagedType.ByValTStr, SizeConst = LF_FACESIZE)]
            public string FaceName;
        }

        [StructLayout(LayoutKind.Sequential)]
        private struct COORD
        {
            public short X;
            public short Y;
        }

        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern IntPtr GetStdHandle(int nStdHandle);

        [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        private static extern bool GetCurrentConsoleFontEx(IntPtr hConsoleOutput, bool bMaximumWindow, ref CONSOLE_FONT_INFO_EX lpConsoleCurrentFontEx);

        [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        private static extern bool SetCurrentConsoleFontEx(IntPtr hConsoleOutput, bool bMaximumWindow, ref CONSOLE_FONT_INFO_EX lpConsoleCurrentFontEx);

        /// <summary>
        /// فونت کنسول را به Consolas تغییر می‌دهد. 
        /// سایز فعلی فونت کاربر را حفظ می‌کند.
        /// </summary>
        public static void SetConsolasFont()
        {
            try
            {
                var hnd = GetStdHandle(STD_OUTPUT_HANDLE);
                if (hnd == IntPtr.Zero || hnd == new IntPtr(-1))
                {
                    return;
                }

                var info = new CONSOLE_FONT_INFO_EX();
                info.cbSize = (uint)Marshal.SizeOf(info);

                // ۱. ابتدا تنظیمات فعلی را می‌خوانیم تا سایز فونت کاربر به هم نریزد
                if (GetCurrentConsoleFontEx(hnd, false, ref info))
                {
                    // ۲. فقط نام فونت را تغییر می‌دهیم
                    info.FaceName = "Consolas";

                    // ۳. تنظیمات جدید را اعمال می‌کنیم
                    SetCurrentConsoleFontEx(hnd, false, ref info);
                }
            }
            catch
            {
                // تغییر فونت یک عملیات حیاتی بیزینسی نیست. اگر ویندوز به هر دلیلی (مثل نداشتن پرمیشن)
                // اجازه این کار را نداد، نباید کل سرویس از کار بیفتد.
            }
        }
    }
}
```

سپس در بخش شروع برنامه خود این خط را قرار دهید:  

```csharp
PersianConsoleWriter.SetEncoding();
Console.SetOut(new PersianConsoleWriter(Console.Out));
Console.SetError(new PersianConsoleWriter(Console.Error));
```

اکنون بصورت خودکار جملات فارسی درست نشان داده می‌شوند.  