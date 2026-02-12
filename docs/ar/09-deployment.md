<div align="center" dir="rtl">

# 09 - النشر (Deployment)

## خط أنابيب CI/CD والبنية التحتية

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مقدمة

تغطي هذه الوثيقة البنية التحتية للنشر، خط أنابيب التكامل المستمر/النشر المستمر (CI/CD)، والاعتبارات التشغيلية لـ SourceShan.

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## مكدس النشر (Deployment Stack)

### نظرة عامة

```
┌─────────────────────────────────────────────────────────────────┐
│                        مكدس النشر (DEPLOYMENT STACK)             │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    منصة Vercel                             │  │
│  │                                                           │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │    Edge     │  │  Serverless │  │   Static    │       │  │
│  │  │  Functions  │  │   Lambda    │  │   Assets    │       │  │
│  │  │(middleware) │  │ (API routes)│  │  (CDN)      │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      طبقة البيانات                         │  │
│  │                                                           │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │  │
│  │  │    MongoDB Atlas    │  │    GitHub Repos     │        │  │
│  │  │   (بيانات المستخدم)  │  │  (بيانات المحفظة)   │        │  │
│  │  └─────────────────────┘  └─────────────────────┘        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### المكونات

| المكون | المزود | الغرض |
|-----------|----------|---------|
| **Edge Functions** | Vercel Edge | برمجيات وسيطة للمصادقة |
| **Serverless Functions** | Vercel | مسارات API |
| **Static Assets** | Vercel CDN | CSS, JS, صور |
| **قاعدة البيانات** | MongoDB Atlas | بيانات المستخدم والمصادقة |
| **تخزين المحتوى** | GitHub | JSON المحفظة والصور |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## خط أنابيب CI/CD

### الإعداد الحالي

توفر Vercel CI/CD تلقائياً:

```
┌─────────────────────────────────────────────────────────────────┐
│                      تدفق النشر (DEPLOYMENT FLOW)                │
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────┐  │
│  │ git push│───►│ Vercel  │───►│  Build  │───►│  Deploy     │  │
│  │         │    │ Webhook │    │         │    │  (Atomic)   │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────┘  │
│                                                                  │
│  الميزات:                                                       │
│  • تلقائي عند كل دفع (push) للفرع الرئيسي                        │
│  • عمليات نشر معاينة (Preview) لطلبات السحب (PRs)               │
│  • تراجع فوري متاح                                              │
│  • عمليات نشر دون توقف (Zero-downtime)                          │
87: └─────────────────────────────────────────────────────────────────┘
```

### محفزات النشر

| المحفز | الإجراء |
|---------|--------|
| الدفع لـ `main` | نشر الإنتاج |
| طلب سحب (Pull Request) | نشر المعاينة |
| يدوي | إعادة نشر من لوحة التحكم |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## تكوين البيئة

### متغيرات البيئة المطلوبة

| المتغير | الوصف | مكان التعيين |
|----------|-------------|-----------|
| `MONGODB_URI` | سلسلة اتصال MongoDB | لوحة تحكم Vercel |
| `JWT_SECRET` | مفتاح توقيع رمز الوصول | لوحة تحكم Vercel |
| `REFRESH_TOKEN_SECRET` | مفتاح توقيع رمز التحديث | لوحة تحكم Vercel |
| `GITHUB_APP_ID` | معرف تطبيق GitHub | لوحة تحكم Vercel |
| `GITHUB_PRIVATE_KEY` | المفتاح الخاص لتطبيق GitHub | لوحة تحكم Vercel |

### استراتيجية البيئة

```
┌─────────────────────────────────────────────────────────────────┐
│                    استراتيجية البيئة                             │
│                                                                  │
│  تطوير (محلي)                                                    │
│  • ملف .env.local (gitignored)                                  │
│  • MongoDB محلي أو Atlas                                         │
│  • بيانات اعتماد GitHub تجريبية                                  │
│                                                                  │
│  معاينة (Vercel)                                                 │
│  • نفس بيئة الإنتاج                                              │
│  • عناوين URL خاصة بكل PR                                        │
│                                                                  │
│  إنتاج (Vercel)                                                  │
│  • MongoDB إنتاج                                                 │
│  • تطبيق GitHub إنتاج                                            │
│  • نطاق مخصص (Custom domain)                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## تكوين البناء (Build Configuration)

### next.config.ts

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    serverActions: {
      bodySizeLimit: '50mb',  // لرفع الصور الكبيرة
    },
  },
};

export default nextConfig;
```

### نصوص البناء (Build Scripts)

```json
{
  "scripts": {
    "dev": "next dev --turbo",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

| النص | الغرض |
|--------|---------|
| `dev --turbo` | تطوير باستخدام Turbopack (تحديث سريع) |
| `build` | بناء الإنتاج |
| `start` | خادم الإنتاج (للاستضافة الذاتية) |
| `lint` | فحوصات ESLint |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## قائمة مراجعة النشر

### ما قبل النشر
- [ ] تعيين جميع متغيرات البيئة في Vercel
- [ ] قائمة السماح لـ IP في MongoDB Atlas تشمل `0.0.0.0/0` (تستخدم Vercel عناوين IP ديناميكية)
- [ ] تطبيق GitHub مثبت على المنظمات المستهدفة
- [ ] تكوين النطاق (إذا كان مخصصاً)

### التحقق ما بعد النشر
- [ ] تدفق تسجيل الدخول يعمل
- [ ] تحديث التوكن يعمل
- [ ] تعديل المحفظة يعمل
- [ ] رفع الصور يعمل
- [ ] لوحة تحكم المسؤول يمكن الوصول إليها للمسؤولين

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## إجراءات التراجع (Rollback)

### عبر لوحة تحكم Vercel
1. الذهاب للوحة تحكم Vercel ← المشروع ← النشرات (Deployments)
2. العثور على آخر نشر يعمل
3. النقر على "..." ← "Promote to Production"
4. تأكيد التراجع

### وقت التراجع
- **فوري** - تحتفظ Vercel بكل أدوات النشر
- **دون توقف** - تبديل ذري للنسخة السابقة

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## المراقبة والرؤية (Observability)

### الحالة الحالية

| القدرة | الحالة |
|------------|--------|
| **سجلات النشر** | ✅ لوحة تحكم Vercel |
| **سجلات وقت التشغيل** | ✅ لوحة تحكم Vercel (محدودة) |
| **تتبع الأخطاء** | ❌ غير منفذ |
| **مقاييس الأداء** | ❌ غير منفذ |
| **مراقبة الجهوزية** | ❌ غير منفذ |

### الإضافات الموصى بها
1. **Sentry** - تتبع الأخطاء والتنبيه
2. **Vercel Analytics** - مراقبة Core Web Vitals
3. **Better Uptime/Pingdom** - مراقبة الجهوزية
4. **LogTail/Axiom** - تجميع السجلات

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## اعتبارات الأمان

### إدارة الأسرار

| السر | طريقة التخزين | التدوير |
|--------|------------|----------|
| `JWT_SECRET` | Vercel env var | يدوي (يوصى بـ 90 يوماً) |
| `REFRESH_TOKEN_SECRET` | Vercel env var | يدوي (يوصى بـ 90 يوماً) |
| `GITHUB_PRIVATE_KEY` | Vercel env var | إعادة توليد تطبيق GitHub |
| `MONGODB_URI` | Vercel env var | تدوير كلمة مرور قاعدة البيانات |

### أمان الشبكة
- MongoDB Atlas: قائمة سماح IP + مصادقة
- GitHub App: مصادقة بالمفتاح الخاص
- Vercel: فرض HTTPS، أمان الحافة

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## اعتبارات التوسع (Scaling)

### الحدود الحالية

| المورد | الحد | ملاحظات |
|----------|-------|-------|
| **Vercel Serverless** | 100 متزامن | خطة Hobby |
| **اتصالات MongoDB** | 100 | مجموعة M0 |
| **GitHub API** | 5000/ساعة | حد معدل التطبيق |

### مسار التوسع
1. **Vercel Pro** - 1000 تنفيذ متزامن
2. **MongoDB M10+** - اتصالات أكثر، موارد مخصصة
3. **GitHub API** - حدود أعلى مع خطة GitHub المدفوعة

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## التعافي من الكوارث

### نقاط الفشل الفردية

| المكون | الخطر | التخفيف |
|-----------|------|------------|
| MongoDB Atlas | فقدان البيانات | نسخ احتياطي آلي (Atlas) |
| GitHub Repos | فقدان المحتوى | تاريخ Git، يمكن استنساخه |
| Vercel | فقدان النشر | إعادة نشر من Git |

### استراتيجية النسخ الاحتياطي

| البيانات | طريقة النسخ الاحتياطي | التكرار |
|------|---------------|-----------|
| بيانات المستخدم | MongoDB Atlas آلي | مستمر |
| محتوى المحفظة | Git (موزع) | كل التزام (commit) |
| تكوين البيئة | موثق | يدوي |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

## ملخص

### مكدس النشر

| الطبقة | التقنية |
|-------|------------|
| **CDN/الحافة** | شبكة Vercel Edge |
| **الحوسبة** | Vercel Serverless (AWS Lambda) |
| **قاعدة البيانات** | MongoDB Atlas |
| **التخزين** | مستودعات GitHub |
| **CI/CD** | نشر تلقائي من Vercel |

### الجاهزية التشغيلية

| القدرة | الحالة |
|------------|--------|
| **نشر آلي** | ✅ جاهز |
| **تراجع فوري** | ✅ جاهز |
| **إدارة البيئة** | ✅ جاهز |
| **المراقبة** | ⚠️ أساسية (تحتاج تحسين) |
| **التنبيه** | ❌ غير منفذ |

<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>

*[← العودة للاختبار والجودة](08-testing-quality.md) | [تابع للنتائج والأثر →](10-results-impact.md)*

</div>
