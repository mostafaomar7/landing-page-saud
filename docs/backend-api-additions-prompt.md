# طلب إضافات صغيرة على Products API (رواف)

## المشكلة
الـ API الحالي ممتاز كأساس، ومش هنغير فيه حاجة: الـ auth ورفع الصور والـ CRUD وشكل الـ response هنشتغل عليهم زي ما هم.

المشكلة إن صفحتنا **صفحة هبوط لمنتج واحد** (خلطة رواف للقولون) مش متجر. المنتج فيه حاجات مالهاش مكان في الـ `Product` الحالي:

1. **3 باقات مش سعر واحد.** المنتج بيتباع كعبوة، أو عبوتين، أو 3 عبوات. كل باقة ليها سعرها، والسعر قبل الخصم، وشارة ("الأكثر مبيعاً" أو "توفير 42%")، وهدية، وواحدة فيهم هي المختارة افتراضياً. الـ `price` الواحد مش كفاية.
2. **المخزون.** الصفحة بتعرض "باقي 7 عبوات فقط"، ومحتاجين نغيّر الرقم ده من الداشبورد.
3. **العرض والعداد.** الصفحة فيها عداد "ينتهي العرض خلال"، ومحتاجين نحدد إمتى العرض بيخلص فعلاً، والعداد يمشي على وقت السيرفر.

**للعلم:** في الفرونت هنستخدم `GET /products/:id` بس للاندنج بيدج. مش هنعمل قسم قائمة منتجات ولا مودال، فمفيش أي تعديل مطلوب على `GET /products`.

## المطلوب
**نفس الـ endpoints ونفس شكل الـ response ونفس أسلوب الـ validation.** كل الإضافات عبارة عن **حقول جديدة على الـ Product**، ومفيش endpoints جديدة.

### 1) حقول جديدة على `Product`

```
packages      Package[]       default []
stockQty      integer | null  default null   (null = مش بنتابع مخزون)
offerTitle    string | null   default null
offerEndsAt   ISO date | null default null
```

`Package` (ينفع يتخزن كـ JSON column على المنتج، مش لازم جدول منفصل):

```
{
  "id": "string",                     // يتولد في الباك إند لو مش مبعوت (عشان الترتيب والتعديل)
  "label": "string",                  // مطلوب، 1–100 حرف        مثال: "3 عبوات"
  "quantity": "integer",              // مطلوب، ≥ 1              مثال: 3
  "price": "number",                  // مطلوب، ≥ 0، رقمين عشريين  مثال: 199
  "compareAtPrice": "number | null",  // اختياري، لازم > price    مثال: 387
  "badgeText": "string | null",       // اختياري، لحد 50 حرف      مثال: "الأكثر مبيعاً"
  "badgeStyle": "red | gold | null",  // مطلوب لو badgeText موجود
  "giftText": "string | null",        // اختياري، لحد 200 حرف
  "isDefault": "boolean"              // باقة واحدة بالظبط true لو في باقات
}
```

### 2) الـ Validation (في `POST` و`PATCH /admin/products`)
- `packages`: لحد 10 باقات.
  - لو القائمة مش فاضية، لازم باقة واحدة بالظبط `isDefault: true`.
  - `packages` لو اتبعتت في الـ PATCH بتستبدل القائمة كلها، زي `images` بالظبط، والترتيب هو ترتيب المصفوفة.
- `compareAtPrice` لو موجود لازم يكون أكبر من `price`.
- `stockQty`: integer ≥ 0 أو null.
- `offerEndsAt`: ISO date صالح أو null.
- `offerTitle`: لحد 150 حرف أو null.
- الأخطاء بنفس الشكل الحالي: 400 ومعاها `details`. للحقول اللي جوه الباقات استخدم path زي `packages.1.price`.

### 3) حقول محسوبة في الـ response (مش بتتخزن)
في كل response فيه `Product`، سواء public أو admin:

- جوه كل package:
  - `discountPercent`: `round((compareAtPrice - price) / compareAtPrice * 100)`، أو `null` لو مفيش `compareAtPrice`.
  - `available`: `stockQty == null || stockQty >= quantity`.
- على المنتج:
  - `isOfferActive`: `offerEndsAt != null && offerEndsAt > now`.
  - `serverTime`: وقت السيرفر الحالي ISO، عشان العداد في الفرونت يتظبط عليه مش على ساعة موبايل العميل.

**`price` الحالي:** سيبه زي ما هو عشان مايتكسرش حاجة. لو المنتج فيه `packages`، رجّع `price` بسعر الباقة الـ default تلقائياً.

### 4) مثال Response لـ `GET /products/:id`

```json
{
  "success": true,
  "data": {
    "id": "…",
    "nameAr": "خلطة رواف للقولون",
    "nameEn": null,
    "descriptionAr": "خلطة رواف للقولون - تركيبة أعشاب طبيعية فاخرة 100% …",
    "descriptionEn": null,
    "price": 199,
    "imageUrl": "https://…/hero.webp",
    "images": ["https://…/hero.webp"],
    "isActive": true,
    "sortOrder": 0,
    "packages": [
      { "id": "p1", "label": "عبوة وحدة", "quantity": 1, "price": 89,  "compareAtPrice": 129, "discountPercent": 31, "badgeText": null,            "badgeStyle": null,  "giftText": "شحن مجاني + هدية ملعقة معيارية",      "isDefault": false, "available": true },
      { "id": "p2", "label": "عبوتين",   "quantity": 2, "price": 149, "compareAtPrice": 258, "discountPercent": 42, "badgeText": "توفير 42%",      "badgeStyle": "red",  "giftText": "شحن مجاني + هدية",                    "isDefault": false, "available": true },
      { "id": "p3", "label": "3 عبوات",  "quantity": 3, "price": 199, "compareAtPrice": 387, "discountPercent": 49, "badgeText": "الأكثر مبيعاً", "badgeStyle": "gold", "giftText": "شحن مجاني + هدية ملعقة + خصم إضافي", "isDefault": true,  "available": true }
    ],
    "stockQty": 7,
    "offerTitle": "عرض اليوم الوطني • خصم 30%",
    "offerEndsAt": "2026-10-01T21:00:00.000Z",
    "isOfferActive": true,
    "serverTime": "2026-09-28T12:00:00.000Z",
    "createdAt": "…",
    "updatedAt": "…"
  }
}
```

### 5) المخزون
الطلبات لسه بتروح **واتساب** ومش بتعدي على الباك إند، فالمخزون **مش مطلوب يتخصم أوتوماتيك**. الأدمن هيعدله بإيده بـ `PATCH /admin/products/:id` ويبعت `{ "stockQty": 5 }`.

## المطلوب تسليمه
1. Migration للحقول الجديدة. المنتجات الموجودة تاخد الـ defaults (`packages: []`، والباقي `null`)، ومفيش حاجة تتكسر.
2. تحديث الـ validation والـ response في `POST` و`PATCH` و`GET` (public و admin).
3. Seed أو تحديث للمنتج الحالي بالـ 3 باقات اللي في المثال فوق، و`stockQty: 7`.
4. تحديث توثيق الـ API بالحقول الجديدة.
