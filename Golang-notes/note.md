go get برای اضافه‌کردن یا تغییر dependencyهای module فعلی است، در حالی که go install برای build و نصب‌کردن executableهای Go، مخصوصاً CLI toolها، استفاده می‌شود.


---

حتماً. این نسخه برای کپی مستقیم داخل نوت مناسبه:

## اجرای `migrate` با Docker به‌جای نصب روی سیستم

در این پروژه، ابزار CLI مربوط به `golang-migrate` مستقیماً روی سیستم Host نصب نشده است.

یعنی به‌جای اینکه مثلاً با:

```bash
go install github.com/golang-migrate/migrate/v4/cmd/migrate@v4.17.0
```

یک executable مثل:

```text
migrate
```

روی سیستم نصب شود، از Docker Image آماده‌ی آن استفاده شده:

```text
migrate/migrate:v4.17.0
```

هر بار که به `migrate` نیاز داریم، یک Container موقت از این Image ساخته می‌شود:

```bash
docker run --rm migrate/migrate:v4.17.0 ...
```

روند کلی:

```text
Docker Image
migrate/migrate:v4.17.0
        ↓
docker run
        ↓
ساخت Container موقت
        ↓
اجرای migrate CLI
        ↓
پایان Command
        ↓
حذف Container به دلیل --rm
```

بنابراین `migrate` روی Host نصب نیست؛ executable آن داخل Docker Image قرار دارد و داخل Container اجرا می‌شود.

### مزیت این روش

نیازی به نصب `migrate` روی سیستم نیست و نسخه‌ی ابزار نیز دقیقاً مشخص است:

```text
v4.17.0
```

در نتیجه اعضای تیم می‌توانند از همان نسخه‌ی مشخص استفاده کنند.

---

## اجرای Commandهای migrate بدون ورود به Container

برای اجرای دستورهای `migrate` لازم نیست ابتدا وارد Container شویم.

مثلاً نیازی نیست این کار را انجام دهیم:

```bash
docker run ...
```

بعد وارد shell کانتینر شویم و داخل آن بنویسیم:

```bash
migrate create ...
```

بلکه Command موردنظر مستقیماً هنگام `docker run` به Container داده می‌شود.

مثلاً:

```bash
docker run --rm \
  --volume "$(pwd)/db:/db" \
  migrate/migrate:v4.17.0 \
  create -ext sql -dir /db/migrations add_users_table
```

در این دستور:

```bash
docker run --rm ...
```

مربوط به Docker است.

این:

```text
migrate/migrate:v4.17.0
```

Docker Image است.

و قسمت بعد از Image:

```bash
create -ext sql -dir /db/migrations add_users_table
```

دستور و آرگومان‌های خود CLI مربوط به `migrate` هستند.

یعنی از نظر مفهومی:

```text
docker run
    ↓
Container ساخته می‌شود
    ↓
داخل Container:
migrate create -ext sql -dir /db/migrations add_users_table
    ↓
Command تمام می‌شود
    ↓
Container حذف می‌شود
```

---

## مثال اجرای Migration

برای اجرای Migrationها نیز دوباره یک Container موقت جدید ساخته می‌شود:

```bash
docker run --rm \
  --network host \
  --volume ./db:/db \
  migrate/migrate:v4.17.0 \
  -path=/db/migrations \
  -database "mysql://root:password@tcp(localhost:3306)/ecomm" \
  up
```

در اینجا دستور اصلی `migrate` تقریباً این است:

```bash
migrate \
  -path=/db/migrations \
  -database "mysql://root:password@tcp(localhost:3306)/ecomm" \
  up
```

ولی به‌جای اینکه `migrate` روی Host نصب شده باشد، این دستور داخل Container اجرا می‌شود.

---

## نکته مهم درباره `docker run`

هر بار که چنین دستوری اجرا می‌شود:

```bash
docker run --rm migrate/migrate:v4.17.0 ...
```

یک **Container جدید** ساخته می‌شود.

مثلاً:

```text
Command اول:
docker run ... create ...
↓
Container A ساخته می‌شود
↓
create اجرا می‌شود
↓
Container A حذف می‌شود


Command دوم:
docker run ... up
↓
Container B ساخته می‌شود
↓
up اجرا می‌شود
↓
Container B حذف می‌شود
```

پس یک Container دائمی برای `migrate` وجود ندارد.

---

## چرا از `docker exec` استفاده نشده؟

`docker exec` زمانی استفاده می‌شود که یک Container از قبل وجود داشته باشد و در حال اجرا باشد:

```bash
docker exec <container> <command>
```

مثلاً MySQL یک Container در حال اجرا دارد:

```text
ecomm-mysql
```

پس می‌توان داخل آن دستور اجرا کرد:

```bash
docker exec ecomm-mysql ...
```

اما Container مربوط به `migrate` موقتی است.

یعنی:

```text
migrate container
↓
ساخته می‌شود
↓
Command اجرا می‌شود
↓
سریعاً متوقف می‌شود
↓
حذف می‌شود
```

پس معمولاً اصلاً Container دائمی‌ای وجود ندارد که بعداً بخواهیم با:

```bash
docker exec
```

داخل آن Command اجرا کنیم.

---

### خلاصه نهایی

```text
migrate روی Host نصب نشده است.

به‌جای:
go install migrate
↓
migrate executable روی Host

از این استفاده شده:
docker run migrate/migrate:v4.17.0
↓
اجرای migrate داخل Container موقت
```

و:

```text
برای هر migrate command:

docker run ...
      ↓
ساخت Container موقت
      ↓
اجرای مستقیم Command داخل Container
      ↓
پایان Command
      ↓
حذف Container (--rm)
```

بنابراین نیازی نیست وارد Container شویم؛ Commandهای `migrate` مستقیماً در همان `docker run` ارسال و اجرا می‌شوند.

---