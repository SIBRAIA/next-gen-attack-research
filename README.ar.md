# اختطاف الثقة عبر طبقات WebView

[**العربية**](./01-cross-layer-webview-trust-hijacking-ar.md) | [**English**](./01-cross-layer-webview-trust-hijacking.md)

**Android WebView · JavaScript Bridge · حدود الثقة Native · Banking Malware · Malvertising · تهديدات AI التكيفية**

يوثق هذا المستودع بحثًا أمنيًا دفاعيًا حول نمط متعدد الطبقات في Android يسمح لمحتوى الويب داخل `WebView` بالعبور إلى وظائف Android الأصلية عبر JavaScript bridges، وruntime script execution، وDOM Instrumentation، ومسارات الصلاحيات.

## اختر اللغة

- 🇸🇦 **العربية:** [البحث الكامل بالعربية](./01-cross-layer-webview-trust-hijacking-ar.md)
- 🇬🇧 **English:** [Full research paper](./01-cross-layer-webview-trust-hijacking.md)

تستخدم النسختان البنية نفسها، ونموذج الأدلة نفسه، وEvidence IDs نفسها، والخط الزمني نفسه، وحدود الإفصاح نفسها.

## السؤال المركزي

> **ماذا يستطيع محتوى WebView أن يفعل داخل تطبيق Android إذا أصبح هذا المحتوى غير موثوق؟**

## لماذا هذا مهم؟

لا يلزم اختراق خادم الموقع الشرعي حتى يصبح تفاعل العميل غير آمن داخل Android client عدائي أو غير موثوق.

```text
محتوى ويب ديناميكي
        ↓
تنفيذ JavaScript
        ↓
Runtime Instrumentation
        ↓
مراقبة DOM / تفاعل المستخدم
        ↓
Web-to-Native Bridge
        ↓
منطق تطبيق Android
        ↓
بيانات / صلاحيات / قدرات الجهاز
```

## النطاق

يختص البحث **بتطبيقات Android وهواتف Android فقط**.

ولا يدعي اختراق الخوادم، أو universal Android RCE، أو منح الصلاحيات تلقائيًا، أو إمكانية الاستغلال عبر كل إعلان، أو إسناد العينة المحللة إلى عائلة malware أو actor محدد.

## نموذج الأدلة

يفصل البحث بين:

- **Sample-confirmed**
- **Platform-confirmed**
- **Threat-intelligence correlation**
- **Research assessment**
- **Not established**

## النتيجة التاريخية

يبين السجل العام أن dynamic Android malware أقدم من SpyNote وSpyMax:

- **2010:** remote-command / botnet-like control
- **2011:** قدرات يتم تنزيلها بعد التثبيت
- **2012:** targeted Android RAT architecture
- **2013:** scripts خبيثة قابلة للتحديث وcommodity RAT tooling
- **2015–2016:** banking overlays وSpyNote
- **2019–2020:** SpyMax consolidation وانتقال الكود
- **2021–2022:** legitimate-site WebView/session abuse وJavaScript/native bridges
- **2024–2026:** MaaS وmalvertising وOn-Device Fraud وDevice Takeover والتوسع الدولي

## نموذج التهديد المستقبلي

يتضمن البحث Forecast محدودًا:

> **من اختطاف الثقة الثابت إلى التلاعب التكيفي بالثقة باستخدام AI.**

وهذا مصنف بوضوح كتوقع بحثي وليس قدرة مثبتة في العينة.

## الإفصاح المسؤول

تم حذف operational payloads والمعرفات الخاصة بالتنفيذ والخرائط الخاصة بالأهداف والتعليمات خطوة بخطوة للاستغلال.

## الإصدار

**v2.0 — الإصدار العام النهائي**

تاريخ النشر: **14 سبتمبر 2026**  
الأدلة محدثة حتى: **13 سبتمبر 2026**

### سلامة الملفات

SHA-256 للنسخة الإنجليزية:

```text
d403356da2a09cde4468f3b150051d94354c217b4180fc0eeffc2a1310d60065
```

SHA-256 للنسخة العربية:

```text
6cd74446c0c21b9168c2c63a53740610ab2007bfe6e9aabca1e0a38a2efd2355
```
