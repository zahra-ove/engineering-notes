بیشتر راجع به Retry در Failed Transactionها تحقیق بشه.

در سیستم‌های دیتابیسی، بعضی خطاهای Transaction موقتی هستند؛ مثل Deadlock، بعضی Lock Timeoutها یا Serialization Failure. در این حالت‌ها می‌توان کل Transaction را دوباره اجرا کرد.

نکات مهم:

- در صورت Deadlock، معمولاً دیتابیس یکی از Transactionها را Rollback می‌کند.
- در Retry نباید فقط Query شکست‌خورده را دوباره اجرا کرد؛ بهتر است کل Transaction از ابتدا اجرا شود.
- Retry باید محدود باشد، مثلاً ۳ بار.
- بین Retryها بهتر است از Backoff استفاده شود.
- Exponential Backoff یعنی فاصله Retryها به‌تدریج بیشتر شود، مثلاً:
  - 50ms
  - 100ms
  - 200ms
- اضافه کردن Jitter باعث می‌شود چند Transaction دوباره دقیقاً همزمان اجرا نشوند.
- همه خطاها Retryable نیستند.

خطاهای معمولاً مناسب برای Retry:
- Deadlock
- بعضی Lock Wait Timeoutها
- Serialization Failure
- بعضی خطاهای موقتی شبکه یا اتصال

خطاهایی که معمولاً نباید Retry شوند:
- Duplicate Key
- Foreign Key Violation
- Invalid Data
- Business Validation Error
- Insufficient Balance

الگوی کلی:

```text
BEGIN
  ↓
Execute Transaction
  ↓
Failure
  ↓
Rollback
  ↓
Is Error Retryable?
  ↓
Backoff + Jitter
  ↓
Retry Entire Transaction
```

نکته مهم این است که Retry راه اصلی جلوگیری از Deadlock نیست. ابتدا باید طراحی Transaction مناسب باشد، مثلاً:

- Transaction کوتاه باشد.
- Lockها با ترتیب ثابت گرفته شوند.
- Lockهای غیرضروری گرفته نشوند.

مثلاً اگر دو حساب باید Lock شوند:

```text
Account 2
Account 10
```

بهتر است همه Transactionها همیشه ابتدا ID کوچک‌تر و سپس ID بزرگ‌تر را Lock کنند:

```text
lock account 2
lock account 10
```

این کار احتمال Deadlock را کاهش می‌دهد.

Retry بیشتر یک Safety Net برای زمانی است که با وجود طراحی مناسب، باز هم به دلیل Concurrency یک خطای موقتی رخ می‌دهد.

در عملیات مالی باید علاوه بر Retry به Idempotency هم توجه شود؛ چون اگر وضعیت Commit نامشخص باشد، اجرای دوباره عملیات ممکن است باعث ثبت دوباره انتقال یا برداشت پول شود.

موضوعاتی که ارزش مطالعه بیشتر دارند:

- Transaction Retry Strategy
- Deadlock Retry
- Exponential Backoff
- Jitter
- Idempotency
- Commit Failure / Unknown Commit State
- Lock Ordering
- Retryable vs Non-Retryable Errors
- Outbox Pattern
- Transaction Retry در Go و MySQL

منابع مناسب برای مطالعه بیشتر:

- MySQL InnoDB Deadlock Documentation
- MySQL Locking Reads
- کتاب Designing Data-Intensive Applications
- بررسی پیاده‌سازی Retry Wrapper برای `database/sql` در Go