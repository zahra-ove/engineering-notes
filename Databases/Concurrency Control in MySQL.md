# Concurrency Control in MySQL

## Pessimistic Locking vs. Optimistic Locking vs. Atomic Update

### مسئلهٔ اصلی: Lost Update

فرض کنیم موجودی یک حساب بانکی ۵۰۰ دلار است.

دو تراکنش هم‌زمان می‌خواهند موجودی را تغییر دهند:

* Transaction A: افزایش ۱۰۰ دلار
* Transaction B: افزایش ۲۰۰ دلار

اگر هر دو موجودی ۵۰۰ را بخوانند، موجودی جدید را در Application محاسبه کنند و بدون کنترل هم‌زمانی ذخیره کنند، ممکن است یکی از تغییرات از بین برود.

```text
Initial Balance = 500

Transaction A        Transaction B
-------------        -------------
READ 500             READ 500

500 + 100 = 600      500 + 200 = 700

UPDATE 600           UPDATE 700

Final Balance = 700 ❌

Expected Balance = 800
```

به این مشکل Lost Update می‌گوییم.

سه تکنیک برای مدیریت این مسئله را بررسی می‌کنیم.

---

# 1. Pessimistic Locking

## مفهوم

در این روش فرض می‌کنیم احتمال تداخل بین تراکنش‌ها وجود دارد؛ بنابراین قبل از خواندن و تغییر داده، آن را قفل می‌کنیم.

در MySQL با موتور InnoDB می‌توانیم از `SELECT FOR UPDATE` استفاده کنیم.

### Example

```sql
BEGIN;

SELECT balance
FROM accounts
WHERE id = 1
FOR UPDATE;

-- Calculate new balance in application

UPDATE accounts
SET balance = 600
WHERE id = 1;

COMMIT;
```

دستور `SELECT FOR UPDATE` یک قفل انحصاری روی رکورد انتخاب‌شده می‌گیرد که در این سناریو تا پایان تراکنش باقی می‌ماند.

تراکنش دیگری که بخواهد همان رکورد را با `FOR UPDATE` قفل کند یا آن را Update کند، باید منتظر آزاد شدن قفل بماند.

### Concurrent Execution

```text
Transaction A             Transaction B
-------------             -------------

BEGIN                     BEGIN

SELECT FOR UPDATE         SELECT FOR UPDATE
      |                         |
   LOCKED                     WAIT
      |
READ 500
      |
500 + 100 = 600
      |
UPDATE 600
      |
COMMIT
      |
RELEASE LOCK
                                |
                              READ 600
                                |
                           600 + 200
                                |
                            UPDATE 800
                                |
                              COMMIT

Final Balance = 800 ✅
```

### Advantages

* از تغییرات هم‌زمان ناسازگار جلوگیری می‌کند.
* برای عملیات Read-Modify-Write و تصمیم‌گیری وابسته به داده کاربرد دارد.
* هنگامی که چند مرحله باید بر اساس وضعیت قفل‌شده اجرا شوند، مفید است.

### Disadvantages

* تراکنش‌های دیگر ممکن است منتظر بمانند.
* نگه داشتن طولانی قفل باعث افزایش Lock Contention می‌شود.
* ترتیب نامناسب قفل‌گیری می‌تواند Deadlock ایجاد کند.

**Key Point:** Pessimistic Locking از ابتدا داده را قفل می‌کند تا هنگام خواندن و تغییر آن، تراکنش دیگری نتواند تغییر ناسازگار انجام دهد.

---

# 2. Optimistic Locking

## مفهوم

در این روش فرض می‌کنیم تداخل بین تراکنش‌ها کم است.

بنابراین هنگام خواندن، قفل انحصاری نمی‌گیریم؛ بلکه موقع Update بررسی می‌کنیم آیا داده از زمان خواندن تغییر کرده است یا خیر.

یکی از روش‌های رایج پیاده‌سازی، استفاده از ستون `version` است.

### Database Structure

```sql
CREATE TABLE accounts (
    id BIGINT PRIMARY KEY,
    balance BIGINT NOT NULL,
    version BIGINT NOT NULL DEFAULT 1
);
```

فرض کنیم وضعیت اولیه:

```text
id      = 1
balance = 500
version = 1
```

### Step 1: Read

```sql
SELECT balance, version
FROM accounts
WHERE id = 1;
```

هر دو تراکنش اطلاعات زیر را می‌خوانند:

```text
balance = 500
version = 1
```

### Step 2: Conditional Update

تراکنش A:

```sql
UPDATE accounts
SET
    balance = 600,
    version = version + 1
WHERE id = 1
  AND version = 1;
```

نتیجه:

```text
Rows Affected = 1

balance = 600
version = 2
```

تراکنش B با اطلاعات قدیمی خود تلاش می‌کند:

```sql
UPDATE accounts
SET
    balance = 700,
    version = version + 1
WHERE id = 1
  AND version = 1;
```

اما مقدار version دیگر ۱ نیست.

نتیجه:

```text
Rows Affected = 0

Optimistic Lock Conflict!
```

### Concurrent Execution

```text
Transaction A             Transaction B
-------------             -------------

READ 500, V1              READ 500, V1
      |                         |
CALCULATE 600             CALCULATE 700
      |                         |
UPDATE WHERE V1                 |
      |                         |
SUCCESS                         |
      |                         |
VERSION = 2               UPDATE WHERE V1
                                |
                              FAILED
                                |
                         Rows Affected = 0
                                |
                         Reload / Retry
```

در صورت تعارض، Application می‌تواند داده را دوباره بخواند و عملیات را با توجه به قواعد تجاری تکرار کند یا خطای Conflict برگرداند.

نکته: MySQL دستور مستقلی به نام `OPTIMISTIC LOCK` ندارد؛ این الگو را با شرط `WHERE` و بررسی نتیجهٔ Update پیاده‌سازی می‌کنیم.

### Advantages

* هنگام خواندن، قفل انحصاری طولانی‌مدت نمی‌گیرد.
* برای سیستم‌هایی با تعارض نوشتن کم کاربرد دارد.
* تغییر داده از زمان خواندن را تشخیص می‌دهد.

### Disadvantages

* نیازمند مدیریت Conflict در Application است.
* در شرایط تعارض زیاد، تلاش‌های مجدد می‌توانند پرهزینه شوند.
* تمام مسیرهای تغییر داده باید قرارداد version را رعایت کنند.

**Key Point:** Optimistic Locking ابتدا اجازهٔ خواندن می‌دهد و هنگام نوشتن بررسی می‌کند آیا داده در این فاصله تغییر کرده است یا خیر.

---

# 3. Atomic Update

## مفهوم

در این روش محاسبه و به‌روزرسانی را مستقیماً در یک دستور SQL انجام می‌دهیم.

به‌جای اینکه ابتدا موجودی را در Application بخوانیم و مقدار جدید را حساب کنیم، محاسبه را به دیتابیس می‌سپاریم.

### Example – MySQL

```sql
UPDATE accounts
SET balance = balance + 100
WHERE id = 1;
```

برای برداشت:

```sql
UPDATE accounts
SET balance = balance - 100
WHERE id = 1;
```

در این روش برای افزایش یا کاهش سادهٔ موجودی به `SELECT FOR UPDATE` جداگانه نیازی نداریم.

خود UPDATE قفل لازم را می‌گیرد و تغییر را بر اساس مقدار فعلی ردیف اعمال می‌کند.

### Concurrent Execution

```text
Initial Balance = 500

Transaction A             Transaction B
-------------             -------------

BEGIN                     BEGIN

UPDATE balance + 100      UPDATE balance + 200
      |                         |
   LOCKED                     WAIT
      |
BALANCE = 600
      |
COMMIT
      |
RELEASE LOCK
                                |
                           CONTINUE
                                |
                           BALANCE = 800
                                |
                              COMMIT

Final Balance = 800 ✅
```

در اینجا افزایش‌های هم‌زمان همدیگر را از بین نمی‌برند.

### Conditional Atomic Update

در سیستم مالی، می‌توانیم شرط کافی بودن موجودی را نیز در همان دستور SQL قرار دهیم.

```sql
UPDATE accounts
SET balance = balance - 100
WHERE id = 1
  AND balance >= 100;
```

اگر موجودی کافی نباشد، دستور هیچ ردیفی را تغییر نمی‌دهد.

در Application باید نتیجهٔ Update بررسی شود.

در این مثال، `RowsAffected = 0` می‌تواند به معنی موجود نبودن حساب یا کافی نبودن موجودی باشد و در صورت نیاز باید این دو وضعیت را جداگانه تشخیص بدهیم.

### Advantages

* جلوگیری از Lost Update در عملیات افزایش یا کاهش ساده.
* حذف Query اضافی برای خواندن مقدار.
* کاهش رفت‌وبرگشت بین Application و Database.
* کوتاه‌تر شدن بازهٔ زمانی عملیات.

### Limitations

* برای تمام منطق‌های پیچیده مناسب نیست.
* به‌تنهایی تمام انواع Deadlock را حذف نمی‌کند.
* جایگزین Transaction برای عملیات چندمرحله‌ای نیست.

**Key Point:** Atomic Update محاسبه را در همان دستور SQL انجام می‌دهد و تغییر را بر اساس مقدار فعلی ردیف اعمال می‌کند.

---

# 4. Final Comparison

| Feature        | Pessimistic      | Optimistic                 | Atomic Update             |
| -------------- | ---------------- | -------------------------- | ------------------------- |
| استراتژی       | قفل قبل از تغییر | بررسی Conflict هنگام تغییر | محاسبه در SQL             |
| ابزار اصلی     | FOR UPDATE       | Version + WHERE            | UPDATE expression         |
| خواندن اولیه   | معمولاً لازم     | لازم                       | برای تغییر ساده لازم نیست |
| رفتار در تعارض | انتظار برای Lock | تشخیص Conflict             | انتظار برای Lock          |
| نیاز به Retry  | ممکن است         | در صورت Conflict           | در برخی خطاها             |
| امکان Deadlock | بله              | بله                        | بله                       |

## نکتهٔ معماری

این سه تکنیک کاملاً جایگزین یکدیگر نیستند.

Pessimistic و Optimistic دو استراتژی کنترل هم‌زمانی هستند.

Atomic Update یک تکنیک برای انجام تغییر در یک دستور SQL است که از مکانیزم قفل‌گذاری خود دیتابیس استفاده می‌کند.

بنابراین Atomic Update نیز در سطح دیتابیس Lock می‌گیرد، حتی اگر خودمان `SELECT FOR UPDATE` ننوشته باشیم.

---

# 5. Important Note for Financial Systems

اگر انتقال پول از حساب A به حساب B شامل چند دستور SQL باشد، هر سه تکنیک همچنان به Transaction مناسب نیاز دارند.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1
  AND balance >= 100;

-- Check RowsAffected = 1.
-- Otherwise, ROLLBACK and stop.

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

-- Check RowsAffected = 1.
-- Otherwise, ROLLBACK and stop.

-- Record transfer and ledger entries
-- with appropriate validation.

COMMIT;
```

این مثال صرفاً ساختار تراکنش را نشان می‌دهد. در سیستم واقعی، باید اعتبارسنجی مبلغ، جلوگیری از انتقال تکراری و ثبت دقیق سوابق مالی نیز انجام شود.

Transaction تضمین می‌کند تغییرات چندمرحله‌ای یا با هم Commit شوند یا Rollback شوند.

Atomic Update از Lost Update در افزایش و کاهش ساده جلوگیری می‌کند.

Idempotency از اجرای تکراری یک درخواست مالی جلوگیری می‌کند.

این سه مفهوم، مسئولیت‌های متفاوتی دارند.

**Final Takeaway:**

* Pessimistic Locking: Lock first, then modify.
* Optimistic Locking: Check for conflicts before saving.
* Atomic Update: Calculate and modify directly in SQL.

Choose the concurrency-control strategy based on business requirements, contention level, and the type of operation.
