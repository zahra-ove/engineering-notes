## How to Reformat Golang Code

در Go برای مرتب و استاندارد کردن کد می‌توانیم از **GoLand** یا ابزار رسمی **gofmt** استفاده کنیم.

### 1. Reformat Code in GoLand

در ویندوز:

`Ctrl + Alt + L`

این شورتکات کد را از نظر indentation، فاصله‌ها و ساختار ظاهری مرتب می‌کند.

### 2. Optimize Imports

برای مرتب کردن و حذف importهای اضافی:

`Ctrl + Alt + O`

### 3. gofmt

`gofmt` ابزار رسمی زبان Go برای فرمت کردن کد است.

فرمت یک فایل:

```bash
gofmt -w main.go
```

فرمت فایل‌های Go در مسیر فعلی:

```bash
gofmt -w .
```

گزینه `-w` یعنی نتیجه‌ی فرمت مستقیماً داخل فایل نوشته شود.

### نکته

Go تأکید زیادی روی یک فرمت استاندارد دارد، بنابراین معمولاً نیازی نیست هر برنامه‌نویس Style شخصی خودش را برای کد تعریف کند.

در GoLand معمولاً این دو شورتکات کافی هستند:

```text
Ctrl + Alt + L  → Reformat Code
Ctrl + Alt + O  → Optimize Imports
```
