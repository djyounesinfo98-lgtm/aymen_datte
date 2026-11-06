# نشر متجر تمور الزيبان — Supabase هو مصدر البيانات الوحيد

## 0) الموقع المنشور على GitHub Pages — تسجيل الطلبات (مهم)
GitHub Pages يعرض ملف `index.html` فقط ولا يشغّل مسارات `/api` الخاصة بـ Next.js.
لذلك تُسجَّل الطلبات من الصفحة المنشورة **مباشرة في Supabase** عبر دالة آمنة داخل قاعدة البيانات `place_order`:
- الأسعار ورسوم التوصيل والمخزون تُحسب داخل Supabase (لا يُوثق بأي سعر من المتصفح).
- المخزون يُخصم داخل معاملة واحدة مع قفل الصفوف: لا بيع زائد.
- الصفحة تستخدم **المفتاح العام فقط** (`sb_publishable_...`)؛ المفتاح السري لا يوجد في الصفحة.

**التفعيل (مرة واحدة):**
1. Supabase ← SQL Editor ← New query ← الصق محتوى `supabase/place_order.sql` ← Run.
   النتيجة المتوقعة في الأسفل: `place_order_ready = true` و `orders_hidden_from_public = true`.
2. أعد توليد الصفحة: `npx tsx --conditions=react-server scripts/build-static.ts` (تحتاج `SUPABASE_URL` و `SUPABASE_PUBLISHABLE_KEY`).
3. ارفع `index.html` الجديد إلى مستودع GitHub (استبدل القديم).

## 1) كيف يتصل المتجر بقاعدة البيانات
كل قراءة وكتابة تمر عبر طبقة واحدة على الخادم: `src/lib/data/store.ts`.

| الأولوية | المصدر | المتغيرات |
|---|---|---|
| 1 | Supabase PostgreSQL مباشر | `SUPABASE_DATABASE_URL` (إن توفرت كلمة مرور القاعدة) |
| 2 | **Supabase REST** (المستخدم حالياً) | `SUPABASE_URL` + `SUPABASE_SECRET_KEY` |
| — | قاعدة محلية | `DATABASE_URL` — **للتطوير فقط**، وممنوعة تماماً مع `REQUIRE_SUPABASE=true` |

- الصور الجديدة تُرفع إلى **Supabase Storage** (المخزن `product-images`، يُنشأ تلقائياً) ويُسجَّل كل ملف في جدول `media`.
- الكود يرفض أي مشروع غير `khwhccamagczshddxznd`.

## 2) متغيرات البيئة في الإنتاج (Secrets — الخادم فقط)
| الاسم | القيمة |
|---|---|
| `SUPABASE_URL` | `https://khwhccamagczshddxznd.supabase.co` |
| `SUPABASE_SECRET_KEY` | المفتاح السري (`sb_secret_...`) من Project Settings ← API Keys |
| `SUPABASE_PROJECT_REF` | `khwhccamagczshddxznd` |
| `SUPABASE_REGION` | `eu-west-1` |
| `REQUIRE_SUPABASE` | `true` |
| `ADMIN_USERNAME` | `admin` |
| `ADMIN_PASSWORD` | كلمة مرور لوحة التحكم |
| `ADMIN_SESSION_SECRET` | نص عشوائي طويل (32+ حرفاً) |

- لا تستخدم البادئة `NEXT_PUBLIC_` لأي من هذه القيم، ولا تضعها في ملفات المشروع المرفوعة.
- في هذه البيئة القيم موجودة في `.env.local` (مستثنى من Git). عند النشر على Vercel انسخها إلى Environment Variables.

## 3) خطوة أمان إلزامية في Supabase (مرة واحدة — مضمّنة أيضاً في supabase/place_order.sql)
المفتاح العام (publishable) يستطيع حالياً قراءة جداول المتجر عبر واجهة Supabase البرمجية، ومنها الطلبات وهواتف الزبائن.
الموقع لا يحتاج هذا الوصول (يستخدم المفتاح السري من الخادم). افتح Supabase ← **SQL Editor** ونفّذ:

```sql
ALTER TABLE public.categories     ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.products       ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.wilayas        ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.orders         ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.order_items    ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.reviews        ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.store_settings ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.media          ENABLE ROW LEVEL SECURITY;
REVOKE ALL ON ALL TABLES    IN SCHEMA public FROM anon, authenticated;
REVOKE ALL ON ALL SEQUENCES IN SCHEMA public FROM anon, authenticated;
```
لا يحذف أي بيانات ولا يؤثر على الموقع. (اختياري لاحقاً: مزامنة تسلسلات المعرّفات — الموقع يتعامل معها تلقائياً.)

## 4) النشر على Vercel
1. ارفع كل ملفات المشروع إلى مستودع **خاص**. (`.env*` و`backups/` و`node_modules` و`.next` مستثناة.)
2. Vercel ← Add New → Project ← اختر المستودع ← أضف متغيرات القسم 2 (Production) ← Deploy.

## 5) اختبار شامل بعد النشر (قراءة/كتابة حقيقية مع تحقق مستقل من Supabase)
```bash
BASE_URL="https://اسم-موقعك.vercel.app" ADMIN_PASSWORD="..." \
SUPABASE_URL="https://khwhccamagczshddxznd.supabase.co" SUPABASE_SECRET_KEY="..." \
python3 scripts/e2e-check.py
```
ينشئ سجلات باسم `E2E-xxxx` ثم يحذفها هي فقط.

## 6) الصفحة الثابتة index.html (اختياري، للعرض فقط)
```bash
npx tsx --conditions=react-server scripts/build-static.ts
```
