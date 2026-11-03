# المطلوب: API لبيانات المنتج في صفحة الهبوط (رواف)

## السياق
عندنا صفحة هبوط (Landing Page) عربي RTL لمنتج واحد حالياً اسمه **"خلطة رواف للقولون"**، والطلبات بتتبعت على واتساب. كل البيانات دلوقتي مكتوبة Static جوه ملف HTML.

المطلوب نحوّل **بيانات المنتج بس** لبيانات ديناميك من Backend + لوحة تحكم، والفرونت هيسحبها من API.

الـ Framework وقاعدة البيانات حسب اختيارك، بس لازم يبقى في:
- Migrations.
- Seed بالبيانات الحالية.
- توثيق للـ API (OpenAPI أو Postman).

## النطاق (Scope)
**داخل النطاق:**
1. بيانات المنتج الأساسية.
2. صور المنتج.
3. الباقات والأسعار.
4. المخزون والعرض (العداد).

**خارج النطاق دلوقتي:** التقييمات، الأسئلة الشائعة، المميزات والمحتوى التسويقي، الطلبات، الدفع، الشحن. هنعملهم في مرحلة تانية. صمّم الجداول بحيث نقدر نربطهم بالمنتج بعدين بسهولة (`product_id`).

---

## 1. بيانات المنتج — جدول `products`

| الحقل | النوع | قواعد | مثال |
|---|---|---|---|
| `id` | PK | | |
| `name` | string(150) | مطلوب | خلطة رواف للقولون |
| `slug` | string(150) | مطلوب، unique، حروف إنجليزي صغيرة و `-` | rawaf-colon |
| `brand` | string(100) | مطلوب | رواف |
| `short_tagline` | string(200) | اختياري | 100% طبيعي • للقولون والمعدة • بدون إضافات |
| `headline` | string(200) | مطلوب | ودّعي النفخة والقولون.. |
| `headline_accent` | string(200) | اختياري | بطريقة طبيعية 100% |
| `description` | text | مطلوب | خلطة رواف للقولون - تركيبة أعشاب طبيعية فاخرة 100% تنظف قولونك، تهدئ القولون العصبي، تخفف الانتفاخ والغازات، وتنظم الهضم. مجربة من الطب الشعبي الأصيل. |
| `net_weight` | decimal(8,2) | مطلوب، > 0 | 150 |
| `weight_unit` | enum: `g`, `kg`, `ml` | مطلوب | g |
| `duration_text` | string(100) | اختياري | تكفي شهر |
| `ingredients_count` | int | اختياري، ≥ 0 | 7 |
| `ingredients_text` | text | اختياري | خلطة رواف للقولون هي توليفة متوازنة من 7 أعشاب طبيعية مجربة للقولون والمعدة. 100% طبيعي • للقولون والمعدة • بدون إضافات، بدون سكر، بدون نكهات، بدون مواد حافظة. تهدئ القولون العصبي، تخفف الانتفاخ والغازات، وتنظم الهضم. |
| `dosage` | string(255) | اختياري | ملعقة صغيرة ممسوحة (5 جرام) مع كوب ماء فاتر قبل الأكل بنص ساعة |
| `storage` | text | اختياري | احفظيه في الكيس المحكم بعيداً عن الرطوبة والحرارة المباشرة، في مكان جاف وبارد. لا يحتاج ثلاجة. |
| `currency` | char(3) | مطلوب، default `SAR` | SAR |
| `status` | enum: `active`, `hidden`, `out_of_stock` | مطلوب، default `hidden` | active |
| `created_at` / `updated_at` | timestamps | | |

> ⚠️ **ملحوظة:** الوزن في الصفحة مكتوب 150 جرام، لكن صورة الكيس مكتوب عليها 250g. خلّي الـ seed بـ 150 لحد ما العميل يأكد الرقم الصح.

---

## 2. صور المنتج — جدول `product_images`

| الحقل | النوع | قواعد |
|---|---|---|
| `id` | PK | |
| `product_id` | FK → products | cascade delete |
| `type` | enum: `hero`, `banner`, `thumbnail`, `gallery` | مطلوب |
| `url` | string | مطلوب |
| `alt` | string(200) | مطلوب (عربي) |
| `width` / `height` | int | تتحسب تلقائياً عند الرفع |
| `sort_order` | int | default 0 |

**القواعد:**
- صورة `hero` واحدة بس لكل منتج. `banner` و `thumbnail` اختياريين، ولو مش موجودين الـ API يرجّع صورة الـ `hero` مكانهم.
- الرفع: `jpg`, `png`, `webp` بحد أقصى 5MB.
- حوّل الصور لـ **WebP** واعمل نسختين على الأقل: عرض 800px وعرض 1600px. رجّعهم في الـ API كـ `srcset`.
- التخزين على S3 أو ما يعادله، أو local storage مع رابط عام.

---

## 3. الباقات والأسعار — جدول `product_packages`

| الحقل | النوع | قواعد | مثال |
|---|---|---|---|
| `id` | PK | | |
| `product_id` | FK → products | cascade delete | |
| `label` | string(100) | مطلوب | 3 عبوات |
| `quantity` | int | مطلوب، ≥ 1 (عدد العبوات اللي بتخصم من المخزون) | 3 |
| `price` | decimal(10,2) | مطلوب، > 0 | 199.00 |
| `compare_at_price` | decimal(10,2) | اختياري، لازم يكون > `price` لو موجود | 387.00 |
| `badge_text` | string(50) | اختياري | الأكثر مبيعاً |
| `badge_style` | enum: `red`, `gold` | مطلوب لو في `badge_text` | gold |
| `gift_text` | string(200) | اختياري | شحن مجاني + هدية ملعقة + خصم إضافي |
| `is_default` | boolean | باقة واحدة بس `true` لكل منتج | true |
| `is_active` | boolean | default true | true |
| `sort_order` | int | default 0 | 3 |

**قواعد البزنس:**
- `discount_percent` **مايتخزنش**. يتحسب في الـ API: `round((compare_at_price - price) / compare_at_price * 100)`، ولو مفيش `compare_at_price` يرجع `null`.
- لما باقة تتعلم `is_default = true`، الباقي من نفس المنتج يرجع `false` تلقائياً (في transaction).
- لو مفيش باقة default، الـ API يعتبر أول باقة active حسب `sort_order` هي الـ default.
- لازم يبقى في باقة active واحدة على الأقل عشان المنتج يبقى `active`.

**بيانات الـ Seed (الحالية في الصفحة):**

| label | quantity | price | compare_at_price | badge_text | badge_style | gift_text | is_default |
|---|---|---|---|---|---|---|---|
| عبوة وحدة | 1 | 89 | 129 | — | — | شحن مجاني + هدية ملعقة معيارية | false |
| عبوتين | 2 | 149 | 258 | توفير 42% | red | شحن مجاني + هدية | false |
| 3 عبوات | 3 | 199 | 387 | الأكثر مبيعاً | gold | شحن مجاني + هدية ملعقة + خصم إضافي | true |

---

## 4. المخزون والعرض

### أعمدة إضافية في `products`

| الحقل | النوع | قواعد | مثال |
|---|---|---|---|
| `stock_qty` | int | ≥ 0، بعدد **العبوات** | 7 |
| `low_stock_threshold` | int | default 10 | 10 |
| `offer_title` | string(150) | اختياري | عرض اليوم الوطني • خصم 30% |
| `offer_starts_at` | datetime (UTC) | اختياري | |
| `offer_ends_at` | datetime (UTC) | اختياري، لازم > `offer_starts_at` | |

### قواعد البزنس
- **الباقة المتاحة:** الباقة تبقى `available = true` بس لو `stock_qty >= quantity`. مثلاً لو المخزون 2، باقة الـ 3 عبوات تظهر بس `available = false`.
- **نفاد المخزون:** لو `stock_qty = 0`، الـ API يرجّع `is_sold_out = true`. الـ `status` نفسه مايتغيرش تلقائياً، الأدمن هو اللي يغيره.
- **المخزون القليل:** `show_low_stock = stock_qty > 0 && stock_qty <= low_stock_threshold`. الفرونت هيعرض "باقي X عبوات" بس لما القيمة دي `true`.
- **العرض الشغال:** `offer.is_active = now between offer_starts_at and offer_ends_at`. لو `offer_starts_at` فاضي، يتحسب من دلوقتي. لو `offer_ends_at` فاضي، مفيش عرض.
- **وقت السيرفر:** الـ API يرجّع `server_time` عشان العداد في الفرونت يتظبط على وقت السيرفر، مش على ساعة جهاز العميل.
- **خصم المخزون:** اعمل service/function جاهزة اسمها `decrementStock(productId, packageId)` تخصم `quantity` بشكل **atomic** (مثلاً `UPDATE ... SET stock_qty = stock_qty - :q WHERE id = :id AND stock_qty >= :q`)، وترجع خطأ لو المخزون مش كفاية. هتتستخدم بعدين في الطلبات، ومش مطلوب endpoint عام لها دلوقتي.
- **سجل المخزون:** أي تعديل يدوي في المخزون من الأدمن يتسجل في جدول `stock_movements`، وفيه الحقول دي:
  - `product_id`
  - `change` (موجب أو سالب)
  - `reason`: `manual`, `order`, `return`
  - `note`
  - `admin_id`
  - `created_at`

---

## الـ API المطلوب

### Public (بدون auth، ومعاه cache 60 ثانية)
`GET /api/products/{slug}` بيرجّع المنتج لو `status != hidden`، وإلا بيرجّع 404.

شكل الـ Response المطلوب بالظبط:

```json
{
  "id": 1,
  "slug": "rawaf-colon",
  "name": "خلطة رواف للقولون",
  "brand": "رواف",
  "short_tagline": "100% طبيعي • للقولون والمعدة • بدون إضافات",
  "headline": "ودّعي النفخة والقولون..",
  "headline_accent": "بطريقة طبيعية 100%",
  "description": "خلطة رواف للقولون - تركيبة أعشاب طبيعية فاخرة 100% ...",
  "net_weight": 150,
  "weight_unit": "g",
  "duration_text": "تكفي شهر",
  "ingredients_count": 7,
  "ingredients_text": "خلطة رواف للقولون هي توليفة متوازنة من 7 أعشاب ...",
  "dosage": "ملعقة صغيرة ممسوحة (5 جرام) ...",
  "storage": "احفظيه في الكيس المحكم ...",
  "currency": "SAR",
  "status": "active",
  "images": {
    "hero":      { "url": "https://cdn.../hero-1600.webp", "srcset": "https://cdn.../hero-800.webp 800w, https://cdn.../hero-1600.webp 1600w", "alt": "خلطة رواف للقولون", "width": 1600, "height": 1067 },
    "banner":    { "...": "نفس الشكل" },
    "thumbnail": { "...": "نفس الشكل" },
    "gallery":   []
  },
  "packages": [
    {
      "id": 3,
      "label": "3 عبوات",
      "quantity": 3,
      "price": 199.00,
      "compare_at_price": 387.00,
      "discount_percent": 49,
      "badge": { "text": "الأكثر مبيعاً", "style": "gold" },
      "gift_text": "شحن مجاني + هدية ملعقة + خصم إضافي",
      "is_default": true,
      "available": true
    }
  ],
  "stock": {
    "qty": 7,
    "is_sold_out": false,
    "show_low_stock": true
  },
  "offer": {
    "title": "عرض اليوم الوطني • خصم 30%",
    "is_active": true,
    "ends_at": "2026-10-01T21:00:00Z"
  },
  "server_time": "2026-09-28T12:00:00Z"
}
```

- `packages` فيها الباقات الـ active بس، ومترتبة حسب `sort_order`.
- الأسعار ترجع كـ number مش string.
- لو العرض مش شغال، `offer` ترجع بـ `is_active: false` و `ends_at: null`.

### Admin (محمي بـ auth: JWT أو Sanctum أو غيره، دور `admin`)

| Method | Endpoint | الوظيفة |
|---|---|---|
| GET | `/api/admin/products` | قائمة المنتجات (مع pagination) |
| POST | `/api/admin/products` | إنشاء منتج |
| GET | `/api/admin/products/{id}` | تفاصيل منتج (كل الحقول حتى الـ hidden) |
| PUT/PATCH | `/api/admin/products/{id}` | تعديل منتج |
| DELETE | `/api/admin/products/{id}` | حذف (soft delete) |
| POST | `/api/admin/products/{id}/images` | رفع صورة (multipart: `file`, `type`, `alt`) |
| PATCH | `/api/admin/images/{id}` | تعديل `alt` / `sort_order` / `type` |
| DELETE | `/api/admin/images/{id}` | حذف صورة (والملفات من الـ storage) |
| POST | `/api/admin/products/{id}/packages` | إضافة باقة |
| PATCH | `/api/admin/packages/{id}` | تعديل باقة |
| DELETE | `/api/admin/packages/{id}` | حذف باقة |
| PATCH | `/api/admin/products/{id}/packages/reorder` | ترتيب الباقات (`[{id, sort_order}]`) |
| POST | `/api/admin/products/{id}/stock` | تعديل المخزون يدوياً (`change`, `note`)، ويتسجل في `stock_movements` |
| GET | `/api/admin/products/{id}/stock-movements` | سجل حركة المخزون |

- رسائل أخطاء الـ validation ترجع بالعربي، وبالشكل ده: `{ "message": "...", "errors": { "field": ["..."] } }` مع status 422.
- أي تعديل من الأدمن يمسح الـ cache بتاع `GET /api/products/{slug}`.
- CORS مسموح لدومين صفحة الهبوط بس.

---

## المطلوب تسليمه
1. Migrations لكل الجداول المذكورة.
2. Seeder بالبيانات الحالية (المنتج والـ 3 باقات و`stock_qty = 7`)، مع رفع صورة المنتج الحالية `assets/product.webp` كصورة `hero`.
3. الـ Endpoints المذكورة مع الـ validation.
4. Tests لقواعد البزنس دي على الأقل:
   - حساب `discount_percent`.
   - باقة default واحدة بس.
   - `available` حسب المخزون.
   - `decrementStock` مايخليش المخزون يبقى سالب مع طلبين في نفس الوقت.
   - `offer.is_active`.
5. توثيق الـ API (OpenAPI أو Postman Collection) فيه أمثلة Request/Response.
6. ملف `README` فيه طريقة التشغيل ومتغيرات البيئة.
