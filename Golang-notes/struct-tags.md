Struct Tags در Go:

db:"field"
→ برای mapping بین struct و database column
→ توسط sqlx / ORM استفاده می‌شود


json:"field"
→ برای تبدیل struct به JSON و برعکس
→ توسط encoding/json استفاده می‌شود


db tag:
Database ↔ Struct

json tag:
HTTP/API JSON ↔ Struct


معمولاً:
Modelهای دیتابیس → db tag

Request/Response API → json tag

گاهی یک struct هر دو را دارد.