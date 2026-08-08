یعنی Makefile یک جور منوی commandهای پروژه است.

Makefile در اصل یک فایل برای تعریف کردن commandهای قابل اجرای پروژه است.

وقتی وارد پروژه‌های Go، Docker، gRPC و CI/CD بشوی، خیلی احتمال دارد چیزی مثل make test، make build، make proto، make lint و make generate ببینی.

make در اصل یک ابزار قدیمی Unix است که برای build automation ساخته شده.

Makefile زبان‌مستقل است.

مثلاً به‌جای اینکه هر بار این‌ها را دستی بزنی:
```
go fmt ./...
go test ./...
go build -o bin/app ./cmd/app
```

می‌توانی در Makefile بنویسی:

```
format:
	go fmt ./...

test:
	go test ./...

build:
	go build -o bin/app ./cmd/app
```


مثلاً در پروژه PHP:

```
test:
	php artisan test

format:
	./vendor/bin/pint

install:
	composer install
```



یک کاربرد خیلی مهم‌تر: ترکیب چند command

اینجا Makefile واقعاً ارزشش را نشان می‌دهد.

مثلاً می‌خواهی محیط development را از صفر آماده کنی:


```
setup:
	docker compose build
	docker compose up -d
	docker compose exec app go mod download
	docker compose exec app go generate ./...
```


حالا چه طور استفاده کنم؟
```
make setup
```
