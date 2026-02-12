<div align="center" dir="rtl">

# 08 - الاختبار والجودة (Testing & Quality)

## نهج الاختبار ومعايير جودة الكود

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مقدمة

تغطي هذه الوثيقة استراتيجية الاختبار، معايير جودة الكود، وممارسات ضمان الجودة في SourceShan.

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## حالة الاختبار الحالية

### تقييم صادق

بناءً على تحليل قاعدة الكود:

| نوع الاختبار | الحالة | الملفات |
|-----------|--------|-------|
| اختبارات الوحدة (Unit) | ❌ غير منفذة | 0 |
| اختبارات التكامل (Integration) | ❌ غير منفذة | 0 |
| اختبارات E2E | ❌ غير منفذة | 0 |

**التغطية: 0%**

### لماذا لا توجد اختبارات (حتى الآن)؟

هذا مشروع لمطور منفرد حيث:
- تم إعطاء الأولوية للسرعة إلى السوق
- كان الاختبار اليدوي أثناء التطوير كافياً
- تم تأجيل البنية التحتية للاختبار

**هذه فجوة معروفة يجب معالجتها.**

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## فلسفة الاختبار

### النهج الموصى به

إذا تم تنفيذ الاختبارات، فإن الاستراتيجية الموصى بها هي:

```
┌─────────────────────────────────────────────────────────────────┐
│                       هرم الاختبار                               │
│                                                                  │
│                          ╱▲╲                                     │
│                         ╱  ╲                                     │
│                        ╱ E2E╲                                    │
│                       ╱──────╲                                   │
│                      ╱        ╲                                  │
│                     ╱ التكامل  ╲                                 │
│                    ╱──────────────╲                              │
│                   ╱                ╲                             │
│                  ╱  اختبارات الوحدة ╲                            │
│                 ╱────────────────────╲                           │
│                                                                  │
│  اختبارات أكثر في الأسفل، أقل في الأعلى                         │
│  اختبارات الأسفل سريعة، اختبارات الأعلى بطيئة                   │
└─────────────────────────────────────────────────────────────────┘
```

### ترتيب الأولويات

1. **اختبارات E2E أولاً** - أعلى عائد على الاستثمار للكود الحالي غير المختبر
2. **اختبارات التكامل** - اختبار مسار API
3. **اختبارات الوحدة** - دوال الأدوات المساعدة، المنطق المعقد

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## تغطية الاختبار الموصى بها

### اختبارات E2E (أولوية 1)

| التدفق | الأولوية | الوصف |
|------|----------|-------------|
| تدفق تسجيل الدخول | حرجة | اسم مستخدم/كلمة مرور ← لوحة التحكم |
| حفظ المحرر | حرجة | تعديل محتوى ← حفظ ← تحقق |
| إدارة المستخدمين (Admin) | عالية | إنشاء/تعديل/حذف المستخدمين |
| تحديث التوكن | عالية | توكن منتهي ← تحديث صامت |

**الأداة الموصى بها:** Playwright

```typescript
// مثال: اختبار تدفق تسجيل الدخول
test('user can login and access editor', async ({ page }) => {
  await page.goto('/login');
  
  await page.fill('[name="username"]', 'testuser');
  await page.fill('[name="password"]', 'testpass');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL('/editor');
  await expect(page.locator('h1')).toContainText('المحرر');
});
```

### اختبارات التكامل (أولوية 2)

| مسار API | الأولوية | حالات الاختبار |
|-----------|----------|------------|
| `/api/auth/login` | حرجة | بيانات صالحة، غير صالحة، حقول ناقصة |
| `/api/auth/refresh` | حرجة | توكن صالح، منتهي، مفقود |
| `/api/portfolio/batch-save` | عالية | ملف واحد، ملفات متعددة، مع الحذف |
| `/api/admin/users` | عالية | عمليات CRUD، فرض RBAC |

**الأداة الموصى بها:** Jest + أدوات اختبار Next.js

### اختبارات الوحدة (أولوية 3)

| الوحدة | دوال للاختبار |
|--------|-------------------|
| `lib/auth.ts` | `signAccessToken`, `verifyRefreshToken` |
| `lib/github.ts` | `normalizePath`, دوال مساعدة |
| `lib/mongodb.ts` | منطق تخزين الاتصال |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## معايير جودة الكود

### تكوين TypeScript

```json
// أبرز نقاط tsconfig.json
{
  "compilerOptions": {
    "strict": true,           // كل الفحوصات الصارمة مفعلة
    "noEmit": true,           // Next.js يعالج التجميع
    "esModuleInterop": true,  // توافق CommonJS/ESM
    "skipLibCheck": true      // تحسين الأداء
  }
}
```

### القواعد المفروضة

| الفئة | القواعد |
|----------|-------|
| **TypeScript** | الوضع الصارم، no-any (تحذير) |
| **React** | قواعد hooks, jsx-key |
| **Next.js** | no-html-link-for-pages, no-sync-scripts |
| **Web Vitals** | قواعد ذات صلة بـ Core Web Vitals |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مقاييس جودة الكود

### من التحليل

| المقياس | القيمة | التقييم |
|--------|-------|------------|
| **أسطر الكود** | ~6,500 | قابل للإدارة |
| **عدد المكونات** | 53 ملف | منظم جيداً |
| **TypeScript** | 100% | تغطية كاملة |
| **ESLint** | مكوّن | نشط |
| **تغطية الاختبار** | 0% | فجوة للمعالجة |

### تنظيم الملفات

```
src/
├── components/          # React components
│   ├── admin/          # Feature: Admin
│   ├── editor/         # Feature: Editor
│   ├── layout/         # Layout components
│   ├── common/         # Shared components
│   └── ui/             # UI primitives
├── context/            # React contexts
├── lib/                # Utility libraries
└── models/             # Database models
```

**التقييم:** تنظيم جيد قائم على الميزات.

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التعامل مع الأخطاء

### النمط المستخدم

```typescript
// نمط API Route
export async function POST(req: Request) {
  try {
    // منطق العمل
    const result = await doSomething();
    return NextResponse.json(result);
    
  } catch (error: unknown) {
    console.error('Operation Error:', error);
    
    const errorMessage = error instanceof Error 
      ? error.message 
      : String(error);
      
    return NextResponse.json(
      { message: 'Operation failed', error: errorMessage },
      { status: 500 }
    );
  }
}
```

### نقاط القوة
- ✅ جميع مسارات API مغلفة بـ try-catch
- ✅ الأخطاء تسجل في وحدة التحكم (Console)
- ✅ استجابات أخطاء مهيكلة
- ✅ استخدام نوع unknown للأخطاء في TypeScript

### الفجوات
- ❌ لا يوجد معالج أخطاء مركزي
- ❌ لا توجد خدمة تتبع أخطاء (Sentry)
- ❌ تسجيل console.error أساسي

### التوصيات
1. إضافة Sentry لتتبع أخطاء الإنتاج
2. إنشاء أداة مساعدة للأخطاء لتنسيق متسق
3. إضافة معرف الطلب (Request ID) لربط السجلات

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## خارطة طريق تحسين الجودة

### المرحلة 1: الأساس (الأسبوع 1-2)
- [ ] إعداد Jest لاختبارات الوحدة
- [ ] إضافة اختبارات لـ `lib/auth.ts`
- [ ] إعداد Playwright لـ E2E
- [ ] إضافة اختبار E2E لتدفق الدخول

### المرحلة 2: المسارات الحرجة (الأسبوع 3-4)
- [ ] E2E: تدفق حفظ المحرر
- [ ] E2E: إدارة مستخدمي المسؤول
- [ ] تكامل: مسارات Auth API
- [ ] تكامل: مسارات Portfolio API

### المرحلة 3: الشمولية (الأسبوع 5-6)
- [ ] اختبارات وحدة لمحرك النماذج
- [ ] اختبارات تكامل لعمليات GitHub
- [ ] إضافة Sentry لتتبع الأخطاء

### المرحلة 4: الأتمتة (الأسبوع 7-8)
- [ ] خط أنابيب CI/CD للاختبارات
- [ ] حدود التغطية
- [ ] Lighthouse CI للأداء

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## ملخص

### الحالة الحالية

| الجانب | الحالة |
|--------|--------|
| **TypeScript** | ✅ وضع صارم |
| **ESLint** | ✅ مكوّن |
| **الاختبار** | ❌ غير منفذ |
| **تتبع الأخطاء** | ❌ غير منفذ |
| **تنظيم الكود** | ✅ مهيكل جيداً |

### الإجراءات ذات الأولوية

1. **إضافة اختبارات E2E** للتدفقات الحرجة (الدخول، المحرر)
2. **إعداد تتبع الأخطاء** مع Sentry
3. **إضافة تحديد المعدل** لنقاط نهاية المصادقة
4. **إنشاء بنية تحتية للاختبار** (Jest + Playwright)

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

<div align="center">

[![Prev](https://img.shields.io/badge/%E2%86%90_%D8%A7%D9%84%D8%A3%D8%AF%D8%A7%D8%A1-5042a5?style=for-the-badge)](07-performance.md) [![Next](https://img.shields.io/badge/Next_%E2%86%92_%D8%A7%D9%84%D9%86%D8%B4%D8%B1-4a45ea?style=for-the-badge)](09-deployment.md)

</div>

</div>