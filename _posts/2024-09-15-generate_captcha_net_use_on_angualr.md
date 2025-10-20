---
title: "ایجاد Captcha در .Net و استفاده در Angular"
categories:
  - Net
tags:
  - net
  - captcha
  - angular
---

برای استفاده از Captcha در فرم‌های خود اگر قصد دارید که خودتان کپچا را بسازید می‌توانید از این کد استفاده کنید.  
توسط این کلاس شما می‌توانید یک عکس کپچا با مشخصات خواسته شده بسازید و سپس آن را در فرم‌های خود نشان دهید.  

```csharp
using System;
using System.Collections.Generic;
using System.Data.SqlClient;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Drawing.Imaging;
using System.IO;
using System.Text;

namespace My
{
    public class CaptchaService : ICaptchaService
    {
        private const string CaptchaPrefix = "captcha";
        private const string TextChars = "ACDEFGHJKLNPQRTUVXYZ2346789";
        private const int CaptchaExpirationTimeInMinutes = 1;

        public CaptchaDto GenerateCaptcha()
        {
            var captchaImage = new CaptchaImage
            {
                FontColor = Color.SteelBlue,
                TextChars = TextChars,
                Width = 120,
                Height = 33,
                TextLength = 6,
                FontWarp = CaptchaImage.FontWarpFactor.Low,
                LineNoise = CaptchaImage.LineNoiseLevel.Low,
                BackgroundNoise = CaptchaImage.BackgroundNoiseLevel.Low,
            };

            var bmp = captchaImage.RenderImage();
            var captchaValue = captchaImage.Text;

            var captchaId = Guid.NewGuid().ToString();

            var key = $"{CaptchaPrefix}-{captchaId}";

            SaveToDb(key, captchaValue, DateTime.Now.Add(TimeSpan.FromMinutes(CaptchaExpirationTimeInMinutes)));


            using (var mem = new MemoryStream())
            {
                bmp.Save(mem, ImageFormat.Png);
                return new CaptchaDto
                {
                    Id = captchaId,
                    Image = $"data:image/png;base64,{Convert.ToBase64String(mem.GetBuffer())}",
                };
            }
        }

        public bool IsCaptchaValid(string id, string value)
        {
            var key = $"{CaptchaPrefix}-{id}";

            return ValidateFromDb(key, value);
        }

        private void SaveToDb(string key, string value, DateTime expirationTime)
        {
            using (var sqlHelper = new SqlHelper(Configs.MyConnectionString))
            {
                var parameters = new List<SqlParameter>
                {
                    new SqlParameter("@Key", key),
                    new SqlParameter("@Value", value),
                    new SqlParameter("@ExpirationTime", expirationTime),
                };

                sqlHelper.ExecuteNonQueryStoredProcedure("[my].[SpSaveCaptcha]", parameters);
            }
        }

        private bool ValidateFromDb(string key, string value)
        {
            using (var sqlHelper = new SqlHelper(Configs.MyConnectionString))
            {
                var parameters = new List<SqlParameter>
                {
                    new SqlParameter("@Key", key),
                    new SqlParameter("@Value", value)
                };

                var foundedData = sqlHelper.FetchDataRowByStoredProcedure("[my].[SpValidateFromDb]", parameters);

                if (foundedData == null || foundedData.HasErrors)
                {
                    throw new UnauthorizedAccessException("not found");
                }

                return foundedData["IsExist"].SafeCast<bool>();
            }
        }
    }

    #region Captcha Generator Class

    public class CaptchaImage
    {
        public CaptchaImage()
        {
            _rand = new Random();
            FontWarp = FontWarpFactor.Low;
            BackgroundNoise = BackgroundNoiseLevel.Low;
            LineNoise = LineNoiseLevel.None;
            _width = 180;
            _height = 50;
            _randomTextLength = 5;
            _randomTextChars = "ACDEFGHJKLNPQRTUVXYZ2346789";
            _fontFamilyName = "";
            FontWhitelist =
                "arial;arial black;comic sans ms;courier new;estrangelo edessa;franklin gothic medium;georgia;lucida console;lucida sans unicode;mangal;microsoft sans serif;palatino linotype;sylfaen;tahoma;times new roman;trebuchet ms;verdana";
            _randomText = GenerateRandomText();
            _generatedAt = DateTime.Now;
            _guid = Guid.NewGuid().ToString();
        }

        /// <summary>
        /// Arithmetic operation to perform in formula
        /// </summary>
        public enum ArithmeticOperation
        {
            Random,
            Addition,
            Substraction
        }

        /// <summary>
        /// Amount of background noise to add to rendered image
        /// </summary>
        public enum BackgroundNoiseLevel
        {
            None,
            Low,
            Medium,
            High,
            Extreme
        }

        /// <summary>
        /// Amount of random font warping to apply to rendered text
        /// </summary>
        public enum FontWarpFactor
        {
            None,
            Low,
            Medium,
            High,
            Extreme
        }

        /// <summary>
        /// Amount of curved line noise to add to rendered image
        /// </summary>
        public enum LineNoiseLevel
        {
            None,
            Low,
            Medium,
            High,
            Extreme
        }

        private readonly DateTime _generatedAt;

        private readonly string _guid;

        private readonly Random _rand;

        private bool _arithmetic;

        private int _arithmeticSum;

        private string _fontFamilyName;

        private int _height;

        private string _randomText;

        private string _randomTextChars;

        private int _randomTextLength;

        private int _width;

        /// <summary>
        /// Returns a GUID that uniquely identifies this Captcha
        /// </summary>
        public string UniqueId => _guid;

        /// <summary>
        /// Returns the date and time this image was last rendered
        /// </summary>
        public DateTime RenderedAt => _generatedAt;

        /// <summary>
        /// Font family to use when drawing the Captcha text. If no font is provided, a random font will be chosen from the font whitelist for each character.
        /// </summary>
        public string Font
        {
            get { return _fontFamilyName; }
            set
            {
                Font font = null;
                try
                {
                    font = new Font(value, 12f);
                    _fontFamilyName = value;
                }
                catch (System.Exception)
                {
                    _fontFamilyName = FontFamily.GenericSerif.Name;
                }
                finally
                {
                    font.Dispose();
                }
            }
        }

        /// <summary>
        /// Amount of random warping to apply to the Captcha text.
        /// </summary>
        public FontWarpFactor FontWarp { get; set; }

        /// <summary>
        /// Amount of background noise to apply to the Captcha image.
        /// </summary>
        public BackgroundNoiseLevel BackgroundNoise { get; set; }

        public LineNoiseLevel LineNoise { get; set; }

        /// <summary>
        /// A string of valid characters to use in the Captcha text.
        /// A random character will be selected from this string for each character.
        /// </summary>
        public string TextChars
        {
            get { return _randomTextChars; }
            set
            {
                _randomTextChars = value;
                _randomText = GenerateRandomText();
            }
        }

        /// <summary>
        /// Number of characters to use in the Captcha text.
        /// </summary>
        public int TextLength
        {
            get { return _randomTextLength; }
            set
            {
                _randomTextLength = value;
                _randomText = GenerateRandomText();
            }
        }

        /// <summary>
        /// Returns the randomly generated Captcha text.
        /// </summary>
        public string Text => _randomText;

        /// <summary>
        /// Width of Captcha image to generate, in pixels
        /// </summary>
        public int Width
        {
            get { return _width; }
            set
            {
                if (value <= 60)
                {
                    throw new ArgumentOutOfRangeException("Width", value,
                        "width must be greater than 60.");
                }

                _width = value;
            }
        }

        /// <summary>
        /// Height of Captcha image to generate, in pixels
        /// </summary>
        public int Height
        {
            get { return _height; }
            set
            {
                if (value <= 30)
                {
                    throw new ArgumentOutOfRangeException("Height", value,
                        "height must be greater than 30.");
                }

                _height = value;
            }
        }

        /// <summary>
        /// A semicolon-delimited list of valid fonts to use when no font is provided.
        /// </summary>
        public string FontWhitelist { get; set; }

        /// <summary>
        /// Background color for the captcha image
        /// </summary>
        public Color BackColor { get; set; } = Color.White;

        /// <summary>
        /// Color of captcha text
        /// </summary>
        public Color FontColor { get; set; } = Color.Black;

        /// <summary>
        /// Color for dots in the background noise
        /// </summary>
        public Color NoiseColor { get; set; } = Color.Black;

        /// <summary>
        /// Color for the background lines of the captcha image
        /// </summary>
        public Color LineColor { get; set; } = Color.Black;

        /// <summary>
        /// Specifies arithmetic operation to use when generating a new captcha
        /// </summary>
        public ArithmeticOperation ArithmeticFunction { get; set; } = ArithmeticOperation.Addition;

        public bool Arithmetic
        {
            get { return _arithmetic; }
            set
            {
                _arithmetic = value;
                _randomTextLength = 7;
                _randomText = GenerateRandomText();
            }
        }

        public int ArithmeticSum => _arithmeticSum;

        /// <summary>
        /// Forces a new Captcha image to be generated using current property value settings.
        /// </summary>
        public Bitmap RenderImage()
        {
            return GenerateImagePrivate();
        }

        private string RandomFontFamily()
        {
            var array = FontWhitelist.Split(';');
            return array[_rand.Next(0, array.Length)];
        }

        private string GenerateRandomText()
        {
            var stringBuilder = new StringBuilder(_randomTextLength);
            var length = _randomTextChars.Length;

            if (!_arithmetic)
            {
                for (var i = 0; i <= _randomTextLength - 1; i++)
                {
                    stringBuilder.Append(_randomTextChars.Substring(_rand.Next(length), 1));
                }
            }
            else
            {
                var num = 0;
                var num2 = 0;

                switch (ArithmeticFunction)
                {
                    case ArithmeticOperation.Addition:
                        num = _rand.Next(2, 99);
                        num2 = _rand.Next(2, 99);
                        stringBuilder.Append($"{num}+{num2}");
                        _arithmeticSum = num + num2;
                        break;

                    case ArithmeticOperation.Substraction:
                        num = _rand.Next(3, 99);
                        num2 = _rand.Next(1, num);
                        stringBuilder.Append($"{num}-{num2}");
                        _arithmeticSum = num - num2;
                        break;
                }
            }

            return stringBuilder.ToString();
        }

        private PointF RandomPoint(int xmin, int xmax, int ymin, int ymax)
        {
            return new PointF(_rand.Next(xmin, xmax), _rand.Next(ymin, ymax));
        }

        private PointF RandomPoint(Rectangle rect)
        {
            return RandomPoint(rect.Left, rect.Width, rect.Top, rect.Bottom);
        }

        private GraphicsPath TextPath(string s, Font f, Rectangle r)
        {
            var stringFormat = new StringFormat();
            stringFormat.Alignment = StringAlignment.Near;
            stringFormat.LineAlignment = StringAlignment.Near;

            var graphicsPath = new GraphicsPath();
            graphicsPath.AddString(s, f.FontFamily, (int)f.Style, f.Size, r, stringFormat);

            return graphicsPath;
        }

        private Font GetFont()
        {
            var emSize = 0f;
            var text = _fontFamilyName;

            if (text == "")
            {
                text = RandomFontFamily();
            }

            switch (FontWarp)
            {
                case FontWarpFactor.None:
                    emSize = Convert.ToInt32(_height * 0.7);
                    break;

                case FontWarpFactor.Low:
                    emSize = Convert.ToInt32(_height * 0.8);
                    break;

                case FontWarpFactor.Medium:
                    emSize = Convert.ToInt32(_height * 0.85);
                    break;

                case FontWarpFactor.High:
                    emSize = Convert.ToInt32(_height * 0.9);
                    break;

                case FontWarpFactor.Extreme:
                    emSize = Convert.ToInt32(_height * 0.95);
                    break;
            }

            return new Font(text, emSize, FontStyle.Bold);
        }

        private Bitmap GenerateImagePrivate()
        {
            var bitmap = new Bitmap(_width, _height, PixelFormat.Format32bppArgb);

            using (var graphics = Graphics.FromImage(bitmap))
            {
                graphics.SmoothingMode = SmoothingMode.AntiAlias;
                var rect = new Rectangle(0, 0, _width, _height);

                Brush brush;

                using (brush = new SolidBrush(BackColor))
                {
                    graphics.FillRectangle(brush, rect);
                }

                var num = 0;
                var num2 = ((double)_width) / _randomTextLength;

                using (brush = new SolidBrush(FontColor))
                {
                    var randomText = _randomText;

                    for (var i = 0; i < randomText.Length; i++)
                    {
                        var c = randomText[i];
                        Font font = null;
                        using (font = GetFont())
                        {
                            var rectangle = new Rectangle(Convert.ToInt32(num * num2), 0, Convert.ToInt32(num2), _height);

                            using (var graphicsPath = TextPath(c.ToString(), font, rectangle))
                            {
                                WarpText(graphicsPath, rectangle);

                                graphics.FillPath(brush, graphicsPath);
                            }
                        }

                        num++;
                    }
                }

                AddNoise(graphics, rect);
                AddLine(graphics, rect);

                return bitmap;
            }
        }

        private void WarpText(GraphicsPath textPath, Rectangle rect)
        {
            var num = 1f;
            var num2 = 1f;

            switch (FontWarp)
            {
                case FontWarpFactor.None:
                    return;

                case FontWarpFactor.Low:
                    num = 6f;
                    num2 = 1f;
                    break;

                case FontWarpFactor.Medium:
                    num = 5f;
                    num2 = 1.3f;
                    break;

                case FontWarpFactor.High:
                    num = 4.5f;
                    num2 = 1.4f;
                    break;

                case FontWarpFactor.Extreme:
                    num = 4f;
                    num2 = 1.5f;
                    break;
            }

            var srcRect = new RectangleF(Convert.ToSingle(rect.Left), 0f, Convert.ToSingle(rect.Width), rect.Height);

            var num3 = Convert.ToInt32(rect.Height / num);
            var num4 = Convert.ToInt32(rect.Width / num);
            var num5 = rect.Left - Convert.ToInt32(num4 * num2);
            var num6 = rect.Top - Convert.ToInt32(num3 * num2);
            var num7 = rect.Left + rect.Width + Convert.ToInt32(num4 * num2);
            var num8 = rect.Top + rect.Height + Convert.ToInt32(num3 * num2);

            if (num5 < 0)
            {
                num5 = 0;
            }

            if (num6 < 0)
            {
                num6 = 0;
            }

            if (num7 > Width)
            {
                num7 = Width;
            }

            if (num8 > Height)
            {
                num8 = Height;
            }

            var pointF = RandomPoint(num5, num5 + num4, num6, num6 + num3);
            var pointF2 = RandomPoint(num7 - num4, num7, num6, num6 + num3);
            var pointF3 = RandomPoint(num5, num5 + num4, num8 - num3, num8);
            var pointF4 = RandomPoint(num7 - num4, num7, num8 - num3, num8);

            var destPoints = new PointF[4]
            {
                pointF,
                pointF2,
                pointF3,
                pointF4
            };

            var matrix = new Matrix();
            matrix.Translate(0f, 0f);

            textPath.Warp(destPoints, srcRect, matrix, WarpMode.Perspective, 0f);
        }

        private void AddNoise(Graphics graphics1, Rectangle rect)
        {
            var num = 0;
            var num2 = 0;
            switch (BackgroundNoise)
            {
                case BackgroundNoiseLevel.None:
                    return;

                case BackgroundNoiseLevel.Low:
                    num = 30;
                    num2 = 40;
                    break;

                case BackgroundNoiseLevel.Medium:
                    num = 18;
                    num2 = 40;
                    break;

                case BackgroundNoiseLevel.High:
                    num = 16;
                    num2 = 39;
                    break;

                case BackgroundNoiseLevel.Extreme:
                    num = 12;
                    num2 = 38;
                    break;
            }

            using (var brush = new SolidBrush(NoiseColor))
            {
                var maxValue = Convert.ToInt32(Math.Max(rect.Width, rect.Height) / num2);

                for (var i = 0; i <= Convert.ToInt32(rect.Width * rect.Height / num); i++)
                {
                    graphics1.FillEllipse(brush, _rand.Next(rect.Width), _rand.Next(rect.Height), _rand.Next(maxValue), _rand.Next(maxValue));
                }
            }
        }

        private void AddLine(Graphics graphics1, Rectangle rect)
        {
            var num = 0;
            var width = 1f;
            var num2 = 0;

            switch (LineNoise)
            {
                case LineNoiseLevel.None:
                    return;

                case LineNoiseLevel.Low:
                    num = 4;
                    width = Convert.ToSingle(_height / 31.25);
                    num2 = 1;
                    break;

                case LineNoiseLevel.Medium:
                    num = 5;
                    width = Convert.ToSingle(_height / 27.7777);
                    num2 = 1;
                    break;

                case LineNoiseLevel.High:
                    num = 3;
                    width = Convert.ToSingle(_height / 25);
                    num2 = 2;
                    break;

                case LineNoiseLevel.Extreme:
                    num = 3;
                    width = Convert.ToSingle(_height / 22.7272);
                    num2 = 3;
                    break;
            }

            var array = new PointF[num + 1];

            using (var pen = new Pen(LineColor, width))
            {
                for (var i = 1; i <= num2; i++)
                {
                    for (var j = 0; j <= num; j++)
                    {
                        array[j] = RandomPoint(rect);
                    }

                    graphics1.DrawCurve(pen, array, 1.75f);
                }
            }
        }
    }

    #endregion
}
```

بخش API :  

```csharp
namespace MyApi
{
    public class MyController : BaseController
    {
        private readonly ICaptchaService _captchaService;

        public MyController()
        {
            _captchaService = new CaptchaService();
        }

        [HttpPost]
        public string Login(string email, string password, string captchaId, string captchaValue)
        {
            if (!_captchaService.IsCaptchaValid(captchaId, captchaValue))
            {
                throw new UnauthorizedAccessException("کد کپچا اشتباه است");
            }

            return "";
        }

        [HttpGet]
        [AllowAnonymous]
        public CaptchaDto GetCaptcha()
        {
            var captcha = _captchaService.GenerateCaptcha();

            return captcha;
        }
    }
}
```

اکنون بصورت زیر می‌توانید از آن استفاده کنید:  

```typescript
  captcha: string = '';
  captchaInput: string = '';
  captchaImage: SafeUrl = '';
  captchaId: string = '';
  requestingCaptcha: boolean = false;

  refreshCaptcha(): void {
    if (this.requestingCaptcha) {
      return
    }

    this.requestingCaptcha = true;
    this.captcha = null;

    this.ss.GetCaptcha().subscribe(item => {
        this.captchaImage = this.imageConverterService.base64ToSafeUrl(item.Image)
        this.captchaId = item.Id;
        this.requestingCaptcha = false;
      },
      err => {
        this.showToast('danger', 'error', err.error);

        this.captchaImage = null;
        this.captchaId = null;
        this.requestingCaptcha = false;
      });
  }
```


```typescript
import {Injectable} from '@angular/core';
import {DomSanitizer, SafeUrl} from "@angular/platform-browser";

@Injectable()
export class ImageConverterService {

  constructor(private sanitizer: DomSanitizer) {
  }

  base64ToSafeUrl(base64: string): SafeUrl {
    return this.sanitizer.bypassSecurityTrustUrl(base64);
  }

  b64toBlob(dataURI: any) {
    let byteString = atob(dataURI.split(',')[1]);
    let arrayBuffer = new ArrayBuffer(byteString.length);
    let uint8Array = new Uint8Array(arrayBuffer);

    for (let i = 0; i < byteString.length; i++) {
      uint8Array[i] = byteString.charCodeAt(i);
    }
    return new Blob([arrayBuffer], {type: 'image/jpeg'});
  }
}
```

```html
<div class="captcha-box">
  <span class="captcha">
    <img
      *ngIf="captchaImage"
      [src]="captchaImage"
      alt="captcha"
      class="captcha"
      draggable="false">
  </span>
  <button
    type="button"
    class="refresh-btn"
    (click)="refreshCaptcha()">
    🔄
  </button>
</div>
```



