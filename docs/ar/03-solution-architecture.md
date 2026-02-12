<div align="center" dir="rtl">

# 03 - معمارية الحل (Solution Architecture)

## كيف تم تصميم SourceShan

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مقدمة

توفر هذه الوثيقة رؤية شاملة لمعمارية SourceShan. سنستكشف تصميم النظام، تفصيل المكونات، تدفق البيانات، والأسباب وراء القرارات المعمارية الرئيسية.

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## الفلسفة المعمارية

قبل الغوص في التفاصيل، من المهم فهم المبادئ التوجيهية:

### 1. أمان Edge-First (Edge-First Security)
تتم المصادقة والتفويض عند الحافة (CDN level) قبل وصول الطلبات إلى الخادم. يؤثر هذا المبدأ على المعمارية بأكملها.

### 2. الدفاع الطبقي (Layered Defense)
تفترض كل طبقة أن الطبقة السابقة ربما تم تجاوزها. يتم فرض الأمان على مستويات Edge و API وقاعدة البيانات.

### 3. المخطط كحقيقة (Schema as Truth)
مخطط قاعدة البيانات هو المصدر الوحيد للحقيقة لتوليد واجهة المستخدم. غيّر المخطط، تتبعه واجهة المستخدم.

### 4. العمليات الذرية (Atomic Operations)
جميع تغييرات الحالة ذرية (Atomic). التحديثات الجزئية غير مقبولة أبداً.

### 5. Serverless-Native
تحتضن المعمارية قيود serverless بدلاً من محاربتها.

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## نظرة عامة على معمارية النظام

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                     │
│                    (Browser / Mobile Browser)                            │
│                                                                          │
│   ┌────────────┐     ┌────────────┐     ┌────────────┐                  │
│   │   Admin    │     │   Client   │     │   Login    │                  │
│   │ Dashboard  │     │   Editor   │     │   Page     │                  │
│   └─────┬──────┘     └─────┬──────┘     └─────┬──────┘                  │
└─────────┼──────────────────┼──────────────────┼─────────────────────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         EDGE LAYER (Vercel Edge)                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     middleware.ts                                  │  │
│  │                                                                   │  │
│  │  • JWT Verification (JOSE)          • Role-Based Access Control  │  │
│  │  • Automatic Token Refresh          • Route Protection           │  │
│  │  • Cookie Management                • Redirect Logic             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER (Node.js Runtime)                   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        API ROUTES                                │    │
│  │                                                                 │    │
│  │  /api/auth/*          /api/admin/*         /api/portfolio/*    │    │
│  │  ├── login            ├── users            ├── batch-save      │    │
│  │  ├── logout           └── [id]             ├── catalog         │    │
│  │  ├── refresh              ├── route.ts     ├── upload          │    │
│  │  └── profile              └── portfolios   └── section         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      SERVICE LAYER                               │    │
│  │                                                                 │    │
│  │  lib/auth.ts      lib/github.ts      lib/mongodb.ts            │    │
│  │  ├── signAccessToken  ├── getOctokit    ├── connectToDatabase  │    │
│  │  ├── signRefreshToken ├── getFileContent├── (singleton pool)   │    │
│  │  └── verifyToken      ├── batchCommit   │                       │    │
│  │                       └── deleteFile    │                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                       DATA LAYER                                 │    │
│  │                                                                 │    │
│  │  models/User.ts                                                 │    │
│  │  ├── username: String                                          │    │
│  │  ├── password: String (hashed)                                 │    │
│  │  ├── role: 'admin' | 'client'                                   │    │
│  │  └── portfolios: [{ id, name, path }]                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│        MongoDB           │   │     GitHub Repos         │
│                          │   │                          │
│  Users Collection        │   │  Per-Client Repos        │
│  ├── _id                 │   │  ├── data/projects.json  │
│  ├── username            │   │  ├── data/about.json     │
│  ├── password (bcrypt)   │   │  ├── public/images/*     │
│  ├── role                │   │  └── (other assets)      │
│  └── portfolios[]        │   │                          │
│      ├── id              │   │  Accessed via:           │
│      ├── name            │   │  • GitHub App API        │
│      └── path (repo)     │   │  • Git Trees API         │
└──────────────────────────┘   └──────────────────────────┘
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## تفصيل المكونات

### 1. طبقة الحافة (Edge Layer - middleware.ts)

**الغرض:** العمل كخط الدفاع الأول، والتحقق من المصادقة، وفرض التحكم في الوصول قبل تنفيذ أي كود من جانب الخادم.

**المسؤوليات الرئيسية:**
- التحقق من JWT باستخدام JOSE (متوافق مع Edge)
- تحديث تلقائي للتوكن عند انتهاء صلاحية رمز الوصول
- حماية المسارات القائمة على الأدوار (Role-based)
- منطق التوجيه للمستخدمين غير المصادق عليهم

**خيار التكنولوجيا:** JOSE بدلاً من jsonwebtoken لأن:
- jsonwebtoken تستخدم واجهات Node.js API غير متوفرة في Edge Runtime
- JOSE مصممة لبيئات JavaScript الحديثة بما فيها Edge

```typescript
// Conceptual flow in middleware
const { payload, newAccessToken } = await verifyOrRefreshToken(req);

if (!payload) {
  // Not authenticated - redirect or 401
  return redirectToLogin(req);
}

if (requiresAdmin(pathname) && payload.role !== 'admin') {
  // Wrong role - 403
  return forbiddenResponse();
}

// Authenticated and authorized - continue
return NextResponse.next();
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

### 2. مسارات API Routes

**الغرض:** التعامل مع منطق العمل وعمليات البيانات للطلبات المصادق عليها.

#### مسارات المصادقة (`/api/auth/*`)

| المسار | الطريقة | الغرض |
|-------|--------|---------|
| `/api/auth/login` | POST | التحقق من الاعتماد، إصدار الرموز |
| `/api/auth/logout` | POST | مسح ملفات تعريف الارتباط للتوكن |
| `/api/auth/refresh` | POST | إصدار رمز وصول جديد باستخدام رمز التحديث |

#### مسارات المسؤول (`/api/admin/*`)

| المسار | الطريقة | الغرض |
|-------|--------|---------|
| `/api/admin/users` | GET | سرد جميع المستخدمين (للمسؤول فقط) |
| `/api/admin/users/[id]` | PUT | تحديث مستخدم (للمسؤول فقط) |

#### مسارات المحفظة (`/api/portfolio/*`)

| المسار | الطريقة | الغرض |
|-------|--------|---------|
| `/api/portfolio/catalog` | GET | الحصول على مخطط المحفظة (catalog.json) |
| `/api/portfolio/batch-save/[id]` | PUT | حفظ ذري (بيانات + صور) |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

### 3. طبقة الخدمة (Service Layer)

#### `lib/auth.ts` - إدارة التوكن

**لماذا نظامين للتحقق؟**
- Edge (middleware.ts) يستخدم JOSE للتحقق
- Node.js (API routes) يستخدم jsonwebtoken للتحقق + التوقيع
- هذا الفصل يسمح بتحسين Edge مع استخدام مكتبة مختبرة للتوقيع

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

#### `lib/github.ts` - تكامل GitHub

**الوظائف الرئيسية:**

| الدالة | الغرض |
|----------|---------|
| `getOctokit` | الحصول على مثيل Octokit مصادق |
| `batchCommit` | عملية ملفات متعددة ذرية |

**GitHub App vs. Personal Access Token:**
- الأساسي: GitHub App authentication (أكثر أماناً، محدد النطاق لكل تثبيت)
- الاحتياطي: Personal Access Token (للتطوير/التصحيح)

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

#### `lib/mongodb.ts` - تجميع الاتصالات

**المشكلة:** Serverless = مثيل جديد لكل طلب = اتصال جديد لكل طلب = استنفاد الاتصالات.

**الحل:** نمط Global Singleton.

```typescript
// Simplified connection pooling pattern
let cached = global.mongoose || { conn: null, promise: null };

async function connectToDatabase() {
  if (cached.conn) return cached.conn;
  // ... connection logic
}
```

**لماذا ينجح هذا:**
- `global` يستمر عبر الاستدعاءات في نفس مثيل Lambda
- الطلب الأول ينشئ الاتصال
- الطلبات اللاحقة تعيد استخدامه
- مثيلات Lambda الجديدة تنشئ اتصالات جديدة (لا مفر منه)

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

### 4. معمارية الواجهة الأمامية (Frontend Architecture)

#### تنظيم المكونات

```
src/components/
├── editor/                   # Editor components
│   └── form-engine/          # نظام النماذج الديناميكي
│       ├── DynamicForm.tsx
│       ├── FieldRenderer.tsx
│       └── ImageInput.tsx
```

#### إدارة الحالة

**استخدام Context API:**

| السياق | الغرض | النطاق |
|---------|---------|-------|
| `UIContext` | إشعارات، نوافذ مشروطة، حالة الملف الشخصي | عالمي |
| `PendingMediaContext` | الصور المرحلية (رفع/حذف) | المحرر |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

### 5. طبقة البيانات (MongoDB)

**قرارات تصميم رئيسية:**

1. **محافظ مضمنة (Embedded Portfolios):** المحافظ مضمنة في مستندات المستخدم بدلاً من مجموعة منفصلة.
   - *ميزة:* استعلام واحد يحصل على المستخدم + المحافظ
   - *ميزة:* تحديثات ذرية

2. **المسار كمرجع:** حقل `path` يشير لمستودع GitHub، لا يخزن المحتوى.
   - MongoDB تخزن *من يملك ماذا*
   - GitHub تخزن *ما هو المحتوى*
   - فصل نظيف للاهتمامات

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مخططات تدفق البيانات

### تدفق المصادقة (Authentication Flow)

```
1. استلام الاعتمادات (username, password)
2. العثور على المستخدم في MongoDB
3. التحقق من كلمة المرور بـ bcrypt
4. توليد access token (15min)
5. توليد refresh token (7d)
6. تعيين الرموز كـ HttpOnly cookies
7. إرجاع دور المستخدم والمحافظ
```

### تدفق الحفظ الدفعي (Portfolio Save Flow)

```
1. استلام FormData (بيانات JSON + صور + مسارات محذوفة)
2. معالجة الصور المعلقة بـ Sharp (تحويل WebP)
3. بناء مصفوفة العمليات:
   - CREATE للصور الجديدة
   - UPDATE لـ data.json
   - DELETE للصور المحذوفة
4. استدعاء batchCommit() الذي:
   أ. يحصل على SHA الالتزام الحالي
   ب. ينشئ Blobs للمحتوى الجديد/المحدث
   ج. يبني شجرة جديدة بكل التغييرات
   د. ينشئ التزاماً واحداً (Commit) يشير للشجرة الجديدة
   هـ. يحدث مرجع الفرع للالتزام الجديد
5. إرجاع النجاح مع SHA الالتزام
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## معمارية الأمان

### الدفاع في العمق (Defense in Depth)

```
LAYER 1: Edge Middleware
• التحقق من التوكن (JOSE)
"خط الدفاع الأول"

LAYER 2: API Route Verification
• إعادة التحقق في العمليات الحساسة
"فحص ثانٍ"

LAYER 3: Role-Based Access Control
• أدوار المسؤول مقابل العميل
"الشخص المناسب، الإجراء المناسب"

LAYER 4: Data Scoping
• الاستعلامات محددة النطاق بمحافظ المستخدم
"المستخدمون يرون بياناتهم فقط"

LAYER 5: HttpOnly Cookies
• الرموز مخزنة في كوكيز HttpOnly
"حتى XSS لا يمكنه سرقة الرموز"
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## اعتبارات التوسع

### حدود المعمارية الحالية

| الجانب | النهج الحالي | الحد | التخفيف |
|--------|-----------------|-------|------------|
| المستخدمين | مجموعة MongoDB واحدة | ~10K مستخدم براحة | فهرسة، ترقيم صفحات |
| المحافظ لكل مستخدم | مصفوفة مضمنة | ~100 لكل مستخدم | ستحتاج مجموعة منفصلة بعدها |
| GitHub API | حدود المعدل (5000/ساعة) | استخدام متزامن كثيف | تخزين مؤقت لمعرف التثبيت |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## ملخص

بُنيت معمارية SourceShan على ثلاث ركائز:

1. **أمان Edge-First** - المصادقة والتفويض قبل الحوسبة
2. **مرونة مدفوعة بالمخطط** - دع هيكل البيانات يقود واجهة المستخدم
3. **عمليات ذرية** - كل التغييرات تنجح أو لا شيء

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<div align="center">

[![Prev: Problem Statement](https://img.shields.io/badge/←_Prev:_Problem_Statement-5042a5?style=for-the-badge)](02-problem-statement.md) [![Next: Key Features](https://img.shields.io/badge/Next:_Key_Features_→-4a45ea?style=for-the-badge)](04-key-features.md)

</div>
