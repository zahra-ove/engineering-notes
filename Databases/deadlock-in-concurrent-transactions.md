# Go Backend Master Class – Lesson 11

## Database Deadlock: PostgreSQL vs. MySQL

### 1. سناریوی انتقال پول

فرض کنیم دو حساب بانکی داریم:

| Account ID | Balance |
| ---------- | ------: |
| 1          |    $500 |
| 2          |    $300 |

جدول `transfers` دو ستون زیر را دارد:

```sql
from_account_id REFERENCES accounts(id)
to_account_id   REFERENCES accounts(id)
```

دو تراکنش هم‌زمان می‌خواهند از حساب شمارهٔ ۱ به حساب شمارهٔ ۲ پول منتقل کنند.

هر تراکنش باید:

1. یک رکورد در `transfers` ایجاد کند.
2. حساب مبدأ را بخواند و قفل کند.
3. موجودی حساب مبدأ را تغییر دهد.
4. حساب مقصد را بخواند و قفل کند.
5. موجودی حساب مقصد را تغییر دهد.
6. تراکنش را Commit کند.

در این مثال برای سادگی، عملیات مربوط به جدول `entries` را نمایش نمی‌دهیم.

---

# Part 1: PostgreSQL

## 2. چرا INSERT باعث ایجاد Lock می‌شود؟

تراکنش اول را در نظر بگیریم:

```sql
BEGIN;

INSERT INTO transfers (
    from_account_id,
    to_account_id,
    amount
)
VALUES (1, 2, 100);
```

در اینجا INSERT روی جدول `transfers` انجام می‌شود.

اما PostgreSQL باید مطمئن شود که حساب‌های شمارهٔ ۱ و ۲ واقعاً در جدول `accounts` وجود دارند و کلیدهای مورد ارجاع تا پایان تراکنش حذف یا به‌شکل ناسازگار تغییر نمی‌کنند.

به همین دلیل، هنگام بررسی Foreign Key، روی ردیف‌های مرجع در جدول `accounts` قفل `FOR KEY SHARE` می‌گیرد.

```text
Transaction A
     |
     v
INSERT INTO transfers
     |
     | Foreign Key Check
     |
     +-- accounts(id=1) → KEY SHARE
     |
     +-- accounts(id=2) → KEY SHARE
```

نکتهٔ مهم:

قفل‌های KEY SHARE با تمام شدن INSERT آزاد نمی‌شوند؛ در این سناریو تا پایان تراکنش باقی می‌مانند.

همچنین KEY SHARE جلوی همهٔ UPDATEها را نمی‌گیرد. هدف اصلی آن جلوگیری از حذف ردیف مرجع یا تغییر ناسازگار کلید آن است.

---

## 3. علت ایجاد Deadlock بین دو تراکنش هم‌زمان

فرض کنیم تراکنش A و B هم‌زمان اجرا می‌شوند.

### مرحلهٔ اول: هر دو INSERT می‌زنند.

Transaction A:

```sql
BEGIN;

INSERT INTO transfers (
    from_account_id,
    to_account_id,
    amount
)
VALUES (1, 2, 100);
```

Transaction B:

```sql
BEGIN;

INSERT INTO transfers (
    from_account_id,
    to_account_id,
    amount
)
VALUES (1, 2, 50);
```

نتیجه:

```text
                  accounts(id=1)
                        |
          +-------------+-------------+
          |                           |
    Transaction A               Transaction B
          |                           |
      KEY SHARE                  KEY SHARE
          |                           |
          +--------- SUCCESS ---------+
```

چرا هر دو موفق شدند؟

چون دو قفل KEY SHARE روی یک ردیف با یکدیگر سازگار هستند.

در این لحظه هر دو تراکنش همچنان باز هستند و قفل‌های خود را نگه داشته‌اند.

### مرحلهٔ دوم: تراکنش A می‌خواهد حساب ۱ را قفل کند.

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

اما تراکنش B هنوز روی حساب ۱ قفل KEY SHARE دارد.

`FOR UPDATE` با `KEY SHARE` متعلق به تراکنش دیگر ناسازگار است.

پس A منتظر آزاد شدن قفل B می‌ماند.

### مرحلهٔ سوم: تراکنش B هم می‌خواهد حساب ۱ را قفل کند.

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

اما تراکنش A نیز هنوز KEY SHARE خود را نگه داشته است.

بنابراین B هم منتظر A می‌ماند.

### شکل نهایی Deadlock

```text
          Transaction A
          -------------
          Holds KEY SHARE
          on account 1
                 |
                 |
        Wants FOR UPDATE
                 |
                 v
           WAITING FOR B
                 |
                 |
             DEADLOCK
                 |
                 |
           WAITING FOR A
                 ^
                 |
        Wants FOR UPDATE
                 |
          Transaction B
          -------------
          Holds KEY SHARE
          on account 1
```

هر تراکنش برای ادامه منتظر قفلی است که تراکنش دیگر باید آزاد کند.

این وضعیت یک چرخهٔ انتظار ایجاد می‌کند.

PostgreSQL چرخه را شناسایی کرده و یکی از تراکنش‌ها را با خطای زیر متوقف می‌کند:

```text
ERROR: deadlock detected
SQLSTATE: 40P01
```

**نکتهٔ کلیدی:** قفل KEY SHARE متعلق به خود تراکنش مانع همان تراکنش نیست. مشکل، قفل ناسازگار متعلق به تراکنش دیگر است.

---

## 4. راه‌حل مدرس در PostgreSQL

مدرس کوئری اولیه را تغییر داد.

Before:

```sql
SELECT *
FROM accounts
WHERE id = $1
FOR UPDATE;
```

After:

```sql
SELECT *
FROM accounts
WHERE id = $1
FOR NO KEY UPDATE;
```

### تفاوت FOR UPDATE و FOR NO KEY UPDATE

| Lock              | سازگاری با KEY SHARE |
| ----------------- | -------------------- |
| FOR UPDATE        | ناسازگار             |
| FOR NO KEY UPDATE | سازگار               |

`FOR NO KEY UPDATE` برای قفل‌کردن ردیف در سناریوی تغییر ستون‌های غیرکلیدی، مثل `balance`، مناسب است.

این دستور خودش تغییر کلید را ممنوع نمی‌کند؛ فقط نوع قفل درخواستی را تعیین می‌کند.

نکتهٔ مهم دیگر:

دو قفل `FOR NO KEY UPDATE` از دو تراکنش متفاوت روی همان ردیف با یکدیگر ناسازگار هستند.

### اجرای مجدد سناریو پس از اصلاح

```text
Transaction A             Transaction B
-------------             -------------

INSERT transfer           INSERT transfer
      |                         |
   KEY SHARE                KEY SHARE
      |                         |
      +--------- OK ------------+
      |                         |
FOR NO KEY UPDATE       FOR NO KEY UPDATE
      |                         |
   LOCK GRANTED             WAITING FOR A
      |
 UPDATE balance
      |
    COMMIT
      |
  Release locks
                                |
                           LOCK GRANTED
                                |
                          UPDATE balance
                                |
                              COMMIT
```

چرا مشکل حل شد؟

چون A برای دریافت `FOR NO KEY UPDATE` دیگر منتظر KEY SHARE متعلق به B نمی‌ماند.

A قفل موردنیاز را می‌گیرد و عملیاتش را ادامه می‌دهد.

B به دلیل ناسازگاری دو قفل NO KEY UPDATE منتظر A می‌ماند.

پس از Commit شدن A، تراکنش B ادامه می‌دهد.

**نتیجه: چرخهٔ انتظار قبلی از بین می‌رود و به‌جای Deadlock، انتظار یک‌طرفه داریم.**

این اصلاح مخصوص مشکل قفل‌گذاری مورد بحث است و تضمین نمی‌کند که تمام انواع Deadlock از بین بروند.

---

# Part 2: MySQL (InnoDB)

## 5. آیا MySQL هم هنگام INSERT قفل می‌گیرد؟

بله.

اگر از موتور InnoDB استفاده کنیم و Foreign Keyها واقعاً تعریف شده باشند، هنگام INSERT در جدول `transfers`، برای بررسی وجود حساب‌های مرجع روی آن‌ها Shared Record Lock می‌گیرد.

مثلاً:

```sql
BEGIN;

INSERT INTO transfers (
    from_account_id,
    to_account_id,
    amount
)
VALUES (1, 2, 100);
```

نتیجه:

```text
INSERT INTO transfers
        |
        | Foreign Key Check
        |
        +-- accounts(id=1) → Shared Lock (S)
        |
        +-- accounts(id=2) → Shared Lock (S)
```

تفاوت اصطلاحات:

* PostgreSQL: `FOR KEY SHARE`
* MySQL InnoDB: `Shared Record Lock (S)`

رفتار این قفل‌ها دقیقاً یکسان نیست، اما هر دو در این سناریو برای محافظت از ردیف مرجع هنگام بررسی Foreign Key استفاده می‌شوند.

MySQL از `SELECT ... FOR UPDATE` پشتیبانی می‌کند، اما دستور مستقیمی به نام `FOR NO KEY UPDATE` ندارد.

بنابراین اگر دو تراکنش ابتدا Shared Lock بگیرند و سپس هر دو درخواست قفل انحصاری روی همان حساب داشته باشند، احتمال وقوع Deadlock وجود دارد.

---

## 6. راه‌حل اول MySQL: تغییر ترتیب عملیات

به‌جای اینکه اول INSERT کنیم و سپس حساب‌ها را قفل کنیم، ترتیب عملیات را تغییر می‌دهیم.

**قانون: ابتدا تمام حساب‌های موردنیاز را با ترتیب ثابت قفل کن؛ سپس INSERT و UPDATE انجام بده.**

برای مثال، همیشه ابتدا حساب با ID کوچک‌تر و سپس حساب با ID بزرگ‌تر قفل شود.

### ترتیب صحیح

```sql
BEGIN;

-- Step 1: Lock account 1

SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;

-- Step 2: Lock account 2

SELECT *
FROM accounts
WHERE id = 2
FOR UPDATE;

-- Step 3: Create transfer

INSERT INTO transfers (
    from_account_id,
    to_account_id,
    amount
)
VALUES (1, 2, 100);

-- Step 4: Update balances

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

این صرفاً نمونه‌ای برای نمایش ترتیب قفل‌گیری است؛ بررسی کافی بودن موجودی و سایر قواعد مالی نیز ضروری است.

### اجرای دو تراکنش هم‌زمان

```text
Transaction A             Transaction B
-------------             -------------

BEGIN                     BEGIN
  |                         |
LOCK account 1           LOCK account 1
  |                         |
SUCCESS                    WAIT
  |
LOCK account 2
  |
SUCCESS
  |
INSERT transfer
  |
UPDATE balances
  |
COMMIT
  |
Release locks
                            |
                         SUCCESS
                            |
                       LOCK account 2
                            |
                       INSERT transfer
                            |
                       UPDATE balances
                            |
                          COMMIT
```

تراکنش B از همان ابتدا برای دریافت قفل حساب ۱ منتظر می‌ماند.

بنابراین هنوز به مرحلهٔ INSERT نرسیده است که بخواهد Shared Lock مربوط به Foreign Key را بگیرد.

تراکنش A هم برای ادامهٔ کار منتظر قفل متعلق به B نیست.

پس چرخهٔ انتظار مورد بحث شکل نمی‌گیرد.

### نکتهٔ مهم دربارهٔ Lock Ordering

اگر انتقال از حساب ۲ به حساب ۱ بود، باز هم باید ابتدا حساب ۱ و سپس حساب ۲ را قفل کنیم.

قانون ما ترتیب IDهاست، نه ترتیب حساب مبدأ و مقصد.

بهتر است در Go، شناسه‌ها را مرتب کرده و سپس با دو Query جداگانه و به ترتیب روی حساب‌ها `FOR UPDATE` اجرا کنیم.

این راه‌حل از Deadlock مشخص این سناریو پیشگیری می‌کند؛ نه از تمام Deadlockهای احتمالی سیستم.

---

## 7. راه‌حل دوم MySQL: Deadlock Detection + Retry

MySQL InnoDB می‌تواند Deadlock را تشخیص دهد.

وقتی Deadlock شناسایی شود، در حالت معمول یکی از تراکنش‌ها Rollback می‌شود و خطای زیر به برنامه برمی‌گردد:

```text
Error 1213
ER_LOCK_DEADLOCK
SQLSTATE 40001
```

مثلاً:

```text
Transaction A       Transaction B
      |                   |
      +---- DEADLOCK -----+
      |                   |
   Continues          ROLLBACK
                          |
                       Error 1213
                          |
                       Retry
                          |
                      NEW BEGIN
```

در Go می‌توانیم این خطا را تشخیص دهیم و تمام عملیات تراکنشی را با یک تراکنش جدید دوباره اجرا کنیم.

بهتر است این مسئولیت در یک Transaction Manager یا Retry Wrapper متمرکز شود.

نکات ضروری:

* تعداد Retryها محدود باشد.
* بین تلاش‌ها از Backoff استفاده شود.
* کل تراکنش تکرار شود، نه فقط آخرین Query.
* Callback باید برای اجرای مجدد ایمن باشد.
* عملیات خارجی مانند ارسال درخواست پرداخت نباید بدون مکانیزم مناسب در Callback قابل‌تکرار قرار بگیرند.

### نکتهٔ حساس در سیستم‌های مالی

Retry خطای Deadlock با Retry یک خطای نامشخص در Commit فرق دارد.

اگر هنگام `COMMIT` ارتباط با دیتابیس قطع شود، ممکن است تراکنش واقعاً Commit شده باشد، ولی پاسخ موفقیت به برنامه نرسیده باشد.

در این حالت نباید کورکورانه تراکنش مالی را تکرار کنیم.

باید با استفاده از شناسهٔ یکتای عملیات، Idempotency و فرایند Reconciliation وضعیت واقعی تراکنش مشخص شود.

---

# Final Summary

| موضوع                       | PostgreSQL              | MySQL InnoDB         |
| --------------------------- | ----------------------- | -------------------- |
| قفل هنگام بررسی Foreign Key | KEY SHARE               | Shared Record Lock   |
| SELECT FOR UPDATE           | دارد                    | دارد                 |
| FOR NO KEY UPDATE           | دارد                    | ندارد                |
| راه‌حل مشکل جلسهٔ ۱۱        | استفاده از قفل سازگارتر | تغییر ترتیب قفل‌گیری |
| تشخیص Deadlock              | دارد                    | دارد                 |
| Retry                       | در سطح برنامه           | در سطح برنامه        |

**نکتهٔ نهایی:** در سناریوی SimpleBank، ابتدا `INSERT INTO transfers` باعث گرفتن قفل‌های مربوط به Foreign Key شد. سپس دو تراکنش برای گرفتن قفل ناسازگار روی حساب‌های مشترک وارد چرخهٔ انتظار شدند.

PostgreSQL با تغییر قفل به `FOR NO KEY UPDATE` این تعارض مشخص را حل کرد. در MySQL می‌توان با قفل‌گیری پیش از INSERT و با ترتیب ثابت از همین سناریو پیشگیری کرد و در کنار آن، برای Deadlockهای باقی‌مانده Retry کنترل‌شده داشت.
