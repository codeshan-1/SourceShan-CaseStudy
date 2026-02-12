<div align="center" dir="rtl">

# 07 - الأداء (Performance)

## استراتيجيات التحسين والنتائج

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مقدمة

الأداء هو اهتمام من الدرجة الأولى في SourceShan. تغطي هذه الوثيقة تحسينات الأداء التي تم تنفيذها، والأسباب وراءها، والنتائج المحققة.

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## فلسفة الأداء

### المبادئ

1. **قم بالقياس أولاً** - لا تحسن ما لم تقم بقياسه
2. **التركيز على المسار الساخن (Hot Path)** - قم بتحسين المسارات الأكثر شيوعاً أولاً
3. **الحافة قبل الأصل (Edge over Origin)** - افعل قدر ما تستطيع عند الحافة
4. **تقليل الرحلات الذهاب والعودة** - قم بعمليات دفعية (Batch) عندما يكون ذلك ممكناً
5. **التحميل الكسول (Lazy Load)** - حمل فقط ما هو مطلوب عندما يكون مطلوباً

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 1: مصادقة الحافة (Edge Authentication)

### التحسين

نقل التحقق من JWT من وقت تشغيل Node.js إلى البرمجيات الوسيطة للحافة (Edge middleware).

### قبل

```
الطلب ← CDN ← Node.js Lambda (بدء بارد) ← التحقق من JWT ← معالجة الطلب
              └───────────────────────────────────────────────────────────┘
                                   ~200-1500ms للبدء البارد
```

### بعد

```
الطلب ← Edge Middleware ← التحقق من JWT ← متابعة أو رفض
             └─────────────────────────────┘
                         ~5-20ms

إذا كان صالحاً:
              ← Node.js Lambda ← معالجة الطلب
```

### الأثر

| المقياس | قبل | بعد |
|--------|--------|-------|
| **زمن استجابة الطلب غير الصالح** | 200-1500ms | <50ms |
| **حوسبة الطلب غير الصالح** | Lambda كاملة | صفر |
| **زمن مصادقة الطلب الصالح** | جزء من الطلب | متوازي مع البدء البارد |

### لماذا ينجح هذا

- تعمل Edge في 30+ موقع عالمي
- لا عقوبة بدء بارد لدوال Edge
- الرموز غير الصالحة لا تصل للخادم أبداً
- يتم التحقق من الرموز الصالحة قبل حتى أن تبدأ Lambda في الإحماء

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 2: تجميع الاتصالات (Connection Pooling)

### التحسين

تخزين اتصالات MongoDB مؤقتاً عبر استدعاءات Lambda داخل نفس الحاوية.

### قبل

```
طلب 1 ← اتصال (100ms) ← استعلام ← استجابة
طلب 2 ← اتصال (100ms) ← استعلام ← استجابة
طلب 3 ← اتصال (100ms) ← استعلام ← استجابة
```

### بعد

```
طلب 1 ← اتصال (100ms) ← استعلام ← استجابة (تخزين الاتصال)
طلب 2 ← استخدام المخزن ← استعلام ← استجابة
طلب 3 ← استخدام المخزن ← استعلام ← استجابة
```

### الأثر

| المقياس | قبل | بعد |
|--------|--------|-------|
| **وقت الاتصال** | كل طلب | أول طلب فقط |
| **العبء لكل طلب** | ~100ms | ~0ms |
| **عدد الاتصالات** | واحد لكل طلب | واحد لكل Lambda |

### التنفيذ

```typescript
let cached = global.mongoose || { conn: null, promise: null };

async function connectToDatabase() {
  if (cached.conn) return cached.conn;
  if (!cached.promise) {
    cached.promise = mongoose.connect(MONGODB_URI);
  }
  cached.conn = await cached.promise;
  return cached.conn;
}
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 3: التزامات GitHub الدفعية (Batch Commits)

### التحسين

دمج عمليات ملفات متعددة في التزام ذري واحد.

### قبل (التزامات لكل ملف)

```
حفظ محفظة بـ 3 صور:
  1. تحديث JSON      ← استدعاء API ← التزام #1
  2. رفع صورة 1      ← استدعاء API ← التزام #2
  3. رفع صورة 2      ← استدعاء API ← التزام #3
  4. رفع صورة 3      ← استدعاء API ← التزام #4
  5. حذف صورة قديمة  ← استدعاء API ← التزام #5

  الإجمالي: 5 استدعاءات API، 5 التزامات
```

### بعد (التزام دفعي)

```
حفظ محفظة بـ 3 صور:
  1. إنشاء blobs لكل المحتوى ← 4 استدعاءات API (متوازية)
  2. بناء شجرة بكل التغييرات ← 1 استدعاء API
  3. إنشاء التزام            ← 1 استدعاء API
  4. تحديث المرجع            ← 1 استدعاء API

  الإجمالي: 7 استدعاءات API (متوازية)، 1 التزام
```

### الأثر

| المقياس | قبل | بعد |
|--------|--------|-------|
| **التزامات لكل حفظ** | N (واحد لكل ملف) | 1 |
| **زمن استجابة API** | متسلسل | متوازي |
| **اتساق البيانات** | في خطر | مضمون |
| **تاريخ الالتزام** | فوضوي | نظيف |

### لماذا ينجح هذا

- يمكن موازاة إنشاء Blob
- التزام واحد يعني عملية ذرية
- استدعاءات API متسلسلة أقل
- تاريخ git نظيف يساعد في التصحيح

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 4: معالجة الصور باستخدام Sharp

### التحسين

تحويل الصور المرفوعة إلى تنسيق WebP بضغط مثالي.

### قبل

```
المستخدم يرفع: photo.jpg (2.5 MB)
النتيجة: photo.jpg مخزنة كما هي (2.5 MB)
```

### بعد

```
المستخدم يرفع: photo.jpg (2.5 MB)
المعالجة: Sharp تحول إلى WebP (جودة: 100)
النتيجة: photo.webp مخزنة (~400 KB)
```

### الأثر

| المقياس | قبل | بعد | التحسن |
|--------|--------|-------|-------------|
| **حجم الصورة** | 2.5 MB | ~400 KB | تقليل 84% |
| **وقت التحميل** | أبطأ | أسرع | متناسب مع الحجم |
| **حجم مستودع Git** | ينمو بسرعة | ينمو ببطء | فائدة طويلة الأجل |

### التنفيذ

```typescript
const webpBuffer = await sharp(Buffer.from(arrayBuffer))
  .webp({ quality: 100 })
  .toBuffer();
```

### المقايضات

- **إيجابي:** ملفات أصغر، تحميل أسرع
- **سلبي:** وقت المعالجة أثناء الحفظ (~100-500ms لكل صورة)
- **القرار:** وقت المعالجة مقبول لأجل أداء أفضل على المدى الطويل

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 5: التخزين المؤقت لمعرف التثبيت (Installation ID Caching)

### التحسين

تخزين معرفات تثبيت تطبيق GitHub مؤقتاً لتقليل استدعاءات API.

### قبل

```
كل طلب API GitHub:
  1. سرد كل التثبيتات ← العثور على المالك المطابق
  2. استخدام معرف التثبيت للمصادقة
  
إذا كان هناك 10 طلبات: 10 عمليات بحث عن التثبيت
```

### بعد

```
أول طلب:
  1. سرد التثبيتات ← العثور على المالك المطابق
  2. تخزين معرف التثبيت مؤقتاً مع TTL

الطلبات اللاحقة:
  1. فحص المخزن المؤقت ← استخدام المعرف المخزن
  
إذا كان هناك 10 طلبات: 1 بحث عن التثبيت + 9 إصابات مخزن مؤقت (cache hits)
```

### الأثر

| المقياس | قبل | بعد |
|--------|--------|-------|
| **استدعاءات API لكل طلب** | +1 (بحث عن تثبيت) | 0 (مخزن مؤقت) |
| **استخدام حد المعدل** | أعلى | أقل |
| **زمن الاستجابة لكل طلب** | +100-200ms | ~0ms |

### التنفيذ

```typescript
const installationCache = new Map<string, { 
  id: number; 
  expiresAt: number 
}>();

async function getInstallationId(owner: string): Promise<number> {
  const cached = installationCache.get(owner);
  if (cached && cached.expiresAt > Date.now()) {
    return cached.id;
  }
  
  const id = await fetchInstallationId(owner);
  installationCache.set(owner, {
    id,
    expiresAt: Date.now() + 3600000 // 1 hour
  });
  return id;
}
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 6: التحميل الكسول للمكونات (Lazy Component Loading)

### التحسين

استخدام استيراد Next.js الديناميكي للمكونات الثقيلة.

### قبل

```javascript
import MainEditor from '@/components/editor/MainEditor';

// يتم تحميل MainEditor حتى لو لم يزر المستخدم صفحة المحرر
```

### بعد

```javascript
import dynamic from 'next/dynamic';

const MainEditor = dynamic(
  () => import('@/components/editor/MainEditor'),
  { loading: () => <EditorSkeleton /> }
);

// يتم تحميل MainEditor فقط عند الحاجة
```

### الأثر

| المقياس | قبل | بعد |
|--------|--------|-------|
| **حجم الحزمة الأولي** | أكبر | أصغر |
| **زمن التفاعل (TTI)** | أبطأ | أسرع |
| **تحميل المحرر** | جزء من الأولي | عند الطلب |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسين 7: تحسين حزمة Framer Motion

### التحسين

استخدام LazyMotion واستيراد الميزات المطلوبة فقط.

### الاستيراد القياسي

```javascript
import { motion, AnimatePresence } from 'framer-motion';
// يحمل حزمة framer-motion بالكامل (~50KB مضغوطة)
```

### الاستيراد المحسن

```javascript
import { LazyMotion, domAnimation, m } from 'framer-motion';

// تغليف التطبيق بـ LazyMotion
<LazyMotion features={domAnimation} strict>
  {children}
</LazyMotion>

// استخدام m بدلاً من motion
<m.div animate={{ opacity: 1 }} />

// الحزمة: ~20KB مضغوطة (تخفيض 60%)
```

### الأثر

| المقياس | قياسي | محسن |
|--------|----------|-----------|
| **حجم الحزمة** | ~50KB | ~20KB |
| **الميزات** | الكل | DOM فقط |
| **جودة الحركة** | نفسها | نفسها |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مقاييس الأداء للتتبع

### الحالة الحالية

بناءً على تحليل الكود، يجب قياس هذه المقاييس:

| الفئة | المقياس | القيمة المتوقعة | للقياس |
|----------|--------|----------------|------------|
| **المصادقة** | زمن تحقق Edge | <20ms | ✅ مقدر |
| **المصادقة** | تكلفة الطلب غير الصالح | $0 | ✅ مصمم |
| **قاعدة البيانات** | معدل إعادة استخدام الاتصال | >90% | ⏳ للقياس |
| **GitHub** | وقت الالتزام الدفعي | <3s | ⏳ للقياس |
| **الصور** | وقت تحويل WebP | <500ms | ⏳ للقياس |
| **التحميل الأولي** | زمن التفاعل | <3s | ⏳ للقياس |

### المراقبة الموصى بها

1. **Vercel Analytics** - مقاييس الأداء الأساسية
2. **Custom Logging** - أوقات تنفيذ مسارات API
3. **MongoDB Atlas** - مقاييس الاتصال والاستعلام
4. **GitHub API** - مراقبة حدود المعدل

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## ميزانية الأداء

### الحدود المقترحة

| المقياس | الهدف | الحد الأقصى |
|--------|--------|-----|
| **استجابة API (p50)** | <200ms | 500ms |
| **استجابة API (p95)** | <500ms | 1000ms |
| **تحميل الصفحة الأولي (LCP)** | <2.5s | 4s |
| **حزمة JavaScript** | <200KB | 300KB |
| **زمن التفاعل** | <3s | 5s |

### التنفيذ

- محلل الحزم في CI
- Lighthouse CI لمقاييس الصفحة
- تنبيهات مخصصة لزمن استجابة API

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التحسينات المستقبلية

### مخطط لها

1. **ذاكرة التخزين المؤقت Redis** - تخزين المخططات التي يتم الوصول إليها بشكل متكرر
2. **CDN للصور** - خدمة الصور من CDN بدلاً من صور GitHub الخام
3. **استجابات التدفق (Streaming)** - تدفق مجموعات البيانات الكبيرة بشكل تزايدي

### قيد النظر

1. **ISR للصفحات الثابتة** - التوليد الثابت التزايدي
2. **تخزين مؤقت للحافة (Edge Caching)** - تخزين استجابات API الشائعة
3. **الترحيل لـ WebP AVIF** - صور أصغر

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## ملخص

| التحسين | الهدف | الآلية |
|--------------|--------|-----------|
| مصادقة الحافة | زمن الاستجابة | نقل التحقق للحافة |
| مسبح الاتصال | عبء قاعدة البيانات | Singleton عالمي |
| التزامات دفعية | كفاءة API | Git Trees API |
| معالجة Sharp | حجم الصورة | تحويل WebP |
| مخزن التثبيت | حدود المعدل | مخزن في الذاكرة |
| التحميل الكسول | حجم الحزمة | استيرادات ديناميكية |
| Framer Motion | حجم الحزمة | LazyMotion |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<div align="center">

[![Prev: Challenges & Solutions](https://img.shields.io/badge/←_Prev:_Challenges_%26_Solutions-5042a5?style=for-the-badge)](06-challenges-solutions.md) [![Next: Testing & Quality](https://img.shields.io/badge/Next:_Testing_%26_Quality_→-4a45ea?style=for-the-badge)](08-testing-quality.md)

</div>
