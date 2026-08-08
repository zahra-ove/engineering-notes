# Protobuf `oneof`

`oneof` زمانی استفاده می‌شود که چند فیلد داریم، اما در هر لحظه فقط **یکی از آن‌ها باید مقدار فعال داشته باشد**.

## مثال

```protobuf
message Laptop {
    oneof weight {
        double weight_kg = 10;
        double weight_lb = 11;
    }
}
```

در این مثال، `weight_kg` و `weight_lb` عضو یک گروه `oneof` هستند.

اگر هر دو فیلد را مقداردهی کنیم، **فقط فیلدی که آخر از همه مقداردهی شده، مقدار خود را حفظ می‌کند** و فیلد قبلی از حالت set خارج می‌شود.

مثلاً:

```go
laptop.WeightKg = 2.5
laptop.WeightLb = 5.5
```

نتیجه:

```text
weight_kg → unset
weight_lb → 5.5
```

اگر برعکس عمل کنیم:

```go
laptop.WeightLb = 5.5
laptop.WeightKg = 2.5
```

نتیجه:

```text
weight_lb → unset
weight_kg → 2.5
```

## مقایسه با فیلدهای معمولی

بدون `oneof`:

```protobuf
double weight_kg = 10;
double weight_lb = 11;
```

هر دو فیلد می‌توانند همزمان مقدار داشته باشند:

```text
weight_kg = 2.5
weight_lb = 5.5
```

اما با `oneof`:

```text
weight
 ├── weight_kg
 └── weight_lb

فقط یکی از آن‌ها می‌تواند active باشد.
```

### نکته

پس عبارت:

> **Only the field assigned last will keep its value**

یعنی:

> در یک گروه `oneof`، آخرین فیلدی که مقداردهی شود، مقدار فعال را نگه می‌دارد و مقدار فیلد قبلی کنار گذاشته می‌شود.
