# اختطاف الثقة عبر طبقات WebView

## رصد وتتبّع المواقع الشرعية، الاحتيال المصرفي، محتوى الأطراف الثالثة، وإساءة استخدام الثقة الأصلية في Android

**مرجع بحث أمني وإفصاح مسؤول**  
**الإصدار:** 2.0 — الإصدار العام النهائي
**التاريخ:** 14 سبتمبر 2026  
**النطاق:** Android WebView · JavaScript Bridge · DOM Instrumentation · مسارات الصلاحيات والقدرات في Android · الارتباط ببرمجيات الاحتيال المصرفي · Malvertising  
**التصنيف:** بحث أمني دفاعي · مرجع تقني للإفصاح المسؤول
**أساس المراجعة:** عينة محللة · توثيق Android الرسمي · استخبارات تهديدات عامة موثقة  
**الأدلة محدثة حتى:** 13 سبتمبر 2026

> **English version:** [01-cross-layer-webview-trust-hijacking.md](./01-cross-layer-webview-trust-hijacking.md)

---

## الملخص التنفيذي

> **حد النطاق:** يختص هذا المنشور **بتطبيقات Android وهواتف Android فقط**. وهو يحلل حدود الثقة على جهة العميل بين WebView وطبقة Android الأصلية. لا يدعي اختراق خادم شركة، أو backend موقع، أو بنية سحابية، أو تطبيق iOS، أو تطبيق سطح مكتب، أو منصة أخرى.

يوثق هذا البحث نمطًا أمنيًا متعدد الطبقات في Android، حيث يمكن لمحتوى ويب معروض داخل `WebView` أن يعبر إلى وظائف Android الأصلية عبر JavaScript Bridge.

العينة المحللة تجمع بين:

```text
محتوى ويب بعيد / ديناميكي
        ↓
تنفيذ JavaScript
        ↓
حقن JavaScript وقت التشغيل
        ↓
مراقبة إدخال DOM / النقرات
        ↓
JavaScript ↔ Native Bridge
        ↓
منطق تطبيق Android
        ↓
تخزين / شبكة / مسارات صلاحيات
```

المشكلة الأساسية ليست API واحدة ضعيفة، بل **تركيب عدة خصائص في سلسلة ثقة واحدة**:

```text
تنقل ديناميكي
    +
JavaScript مفعّل
    +
addJavascriptInterface(...)
    +
دوال Native حساسة
    +
حقن JavaScript وقت التشغيل
    +
عزل Origin ضعيف
    +
مسارات Native مرتبطة بالصلاحيات
```

ينتج عن ذلك **فشل في حد الثقة بين Web وNative**.

لا يلزم اختراق الموقع الشرعي نفسه حتى تصبح جلسة المستخدم غير آمنة. فإذا فُتح موقع مصرفي أو تسجيل دخول أو دفع أو مراسلة أو إدارة حساب داخل WebView خبيث أو غير موثوق بما يكفي، يستطيع التطبيق المضيف إضافة منطق رصد إلى الصفحة محليًا، ومراقبة تفاعل المستخدم، ونقل بيانات مختارة إلى كود Android الأصلي.

يصف البحث هذا النمط بأنه:

> **Cross-Layer WebView Trust Hijacking — اختطاف الثقة عبر طبقات WebView**

أو:

> **رصد وتتبّع موقع شرعي على جهة العميل من داخل WebView في Android مزود بجسر Native.**

ولا ينبغي وصفه تلقائيًا بأنه اختراق لخادم الموقع، أو `RCE` على الخادم، أو اختراق للموقع الشرعي لدى جميع المستخدمين.

---

## نموذج الأدلة والمصطلحات

> **تتبّع الأدلة:** خريطة الادعاءات إلى الأدلة وتصنيفات الإثبات موجودة في **Appendix A — Evidence & Claim Matrix** و**Appendix B — Evidence Boundaries**. تُحتفظ معرفات العينة الخاصة ومواضع المصدر التفصيلية بشكل خاص.

يفصل هذا المنشور بين أربع فئات رئيسية من الأدلة:

| فئة الدليل | المعنى |
|---|---|
| **Sample-confirmed** | ملاحظ مباشرة داخل المصدر/العينة المحللة. |
| **Platform-confirmed** | سلوك أمني موثق رسميًا من Android / Google. |
| **Threat-intelligence correlation** | سلوك موثق مستقلًا في أبحاث malware عامة؛ يستخدم للمقارنة والسياق وليس لإثبات هوية المؤلف. |
| **Research assessment** | استنتاج بحثي مبني على جمع عدة أنواع من الأدلة وموسوم بوضوح كاستنتاج. |

مصطلح **Cross-Layer WebView Trust Hijacking** هو مصطلح بحثي مستخدم في هذه الورقة لوصف نمط حد الثقة الذي تمت ملاحظته. وهو **ليس** اسم CVE رسميًا، ولا تصنيف Android رسميًا، ولا taxonomy صناعية معتمدة.

تدعم العينة الاستنتاج المعماري الآتي:

> **عندما يستطيع WebView تحميل محتوى ديناميكي، وتشغيل/حقن JavaScript، وكشف دوال Native، ومراقبة تفاعل DOM، وبدء مسارات Native، فإنه يتحول إلى حد أمني قد يؤدي فشله إلى نقل حالة يسيطر عليها الويب إلى نطاق ثقة تطبيق Android.**

### سياق العينة وحدود الإسناد

يحتوي المصدر المحلل على سلوك متسق مع أدوات هجومية في Android، بما في ذلك جمع إدخال DOM، تمرير البيانات إلى Native، مسارات مرتبطة بالصلاحيات، وكود إرسال عبر اتصال شبكي.

لكن لقطة المصدر وحدها **لا تثبت**:

```text
جهة تهديد محددة
حملة محددة
تاريخ تشغيل فعلي
مجموعة ضحايا
صلة مؤكدة بعائلة malware مسماة
```

هذه المسائل تتطلب أدلة attribution مستقلة. لذلك تعامل المقارنات مع عائلات malware في بقية البحث على أنها **Correlation** وليست إثبات هوية أو مؤلف مشترك.

---

## منهجية التحليل وحدود التحقق

تمت مراجعة العينة غير العامة عبر **تحليل ثابت للمصدر (Static Source Analysis)**. يعيد البحث بناء المسارات الأمنية من إعدادات WebView، وJavaScript Instrumentation، ودوال الجسر Native، ومنطق الصلاحيات، ومسارات المعالجة/الإرسال الأصلية.

ما لم يُذكر خلاف ذلك، فإن **Sample-confirmed** تعني أن الخاصية أو المسار موجود في المصدر الذي تمت مراجعته. ولا يعني ذلك تلقائيًا أن كل مسار نُفذ ديناميكيًا، أو أن كل backend كان قابلًا للوصول، أو أن كل نتيجة أمنية أُعيد إنتاجها فعليًا على جهاز.

```text
وجود مسار في المصدر
        ≠
إثبات إمكانية الوصول له وقت التشغيل
        ≠
إعادة إنتاج استغلال كامل من طرف إلى طرف
        ≠
إثبات شدة ثغرة خاصة بمنتج محدد
```

الاختبارات الديناميكية، والتقاط الشبكة، واختبار إصدارات الأجهزة، والتحقق الخاص بهدف حي، تُترك للاختبارات المصرح بها وللإفصاح المسؤول.

---

## سياسة الإفصاح التقني العام

تستخدم النسخة العامة عمدًا **مقتطفات عامة ومجردة وأسماء APIs قياسية** بدلًا من نشر الكود غير العام.

وتركز فقط على ما يحتاجه المدافعون لفهم المشكلة:

```text
حد الثقة داخل WebView
تركيبات APIs الخطرة
مسار البيانات
مسار تنسيق الصلاحيات
الأثر على هاتف Android
فرص الاكتشاف
وسائل التخفيف
```

ولا تنشر المصدر الكامل، أو identifiers الخاصة بالتنفيذ، أو بنية المشغل، أو خرائط أهداف محددة، أو بصمات provenance.

الهدف هو توثيق التقنية وتمكين الإصلاح دون إعادة توزيع تطبيق هجومي غير منشور.

---

# 1. النتيجة الأساسية

يفعّل WebView محل التحليل JavaScript ويكشف كائن Java للصفحة:

```java
webView.getSettings().setJavaScriptEnabled(true);

webView.addJavascriptInterface(
    new NativeBridge(),
    "Android"
);
```

مفاهيميًا، تحصل JavaScript على كائن مشابه لـ:

```javascript
window.Android
```

وجود `addJavascriptInterface()` وحده ليس ثغرة تلقائيًا.

التأثير الأمني يعتمد على أسئلة مثل:

```text
ما الصفحات القادرة على الوصول للجسر؟
ما الإطارات frames القادرة على الوصول له؟
ما وظائف Native المكشوفة؟
هل يمكن التنقل خارج Origin موثوق؟
هل يمكن حقن scripts وقت التشغيل؟
هل تستطيع أحداث DOM بدء سلوك Native حساس؟
هل يمكن أن تغادر البيانات الناتجة الجهاز؟
```

في التصميم الذي تمت مراجعته، تلتقي هذه العناصر معًا.

---

# 2. التنقل الديناميكي ليس تحققًا من Origin

تقبل العينة عناوين شبكة تشمل:

```text
https://
http://
```

وتمررها إلى:

```java
webView.loadUrl(uri);
```

التحقق من الـscheme ليس سياسة ثقة للـorigin.

فكل الأمثلة التالية تجتاز فحصًا بدائيًا يبدأ بـHTTPS:

```text
https://trusted.example
https://partner.example
https://third-party.example
https://attacker-controlled.example
```

ينبغي لـWebView مزود بجسر حساس أن يبني قرارات الثقة على الأقل على:

```text
scheme
host
port
navigation state
frame origin
```

وألا يبقي Native Bridge حساسًا مكشوفًا أثناء تنقل عشوائي.

المسار في المصدر **يقبل ويحاول تحميل** كلًا من `http://` و`https://`. لكن نجاح HTTP cleartext على جهاز/بناء معين يعتمد أيضًا على Manifest وNetwork Security Configuration وtarget SDK وسياسات المنصة. لذلك تثبت العينة قبول HTTP على مستوى التطبيق، لا إمكانية cleartext على كل بيئات Android.

---

# 3. Native Bridge حساس أمنيًا

تحتوي الواجهة المحللة على عدة دوال موسومة بـ:

```java
@JavascriptInterface
```

ومن القدرات التمثيلية:

```text
استقبال بيانات من الويب
تمرير بيانات داخل Native
بدء مسار متعلق بالصلاحيات
معالجة بيانات داخلية
```

تمثيل مبسط:

```java
class NativeBridge {

    @JavascriptInterface
    public void receiveWebData(String value) {
        // بيانات قادمة من سياق الويب تدخل إلى Native.
    }

    @JavascriptInterface
    public void requestNativeAction(String request) {
        // مدخل قادم من الويب يؤثر على سلوك Native.
    }
}
```

وبذلك يصبح مسار الثقة:

```text
JavaScript داخل الويب
      ↓
Native Bridge
      ↓
منطق التطبيق
      ↓
تخزين / شبكة / Android UI
```

والخاصية الأمنية الجوهرية هي:

> **حالة يتحكم بها الويب تصبح مدخلًا لمنطق التطبيق الأصلي.**

---

# 4. حقن JavaScript وقت التشغيل

تحتوي العينة على مسار مكافئ لـ:

```java
webView.evaluateJavascript(runtimeScript, null);
```

معماريًا:

```text
بيانات Runtime
    ↓
فك/تحويل
    ↓
evaluateJavascript(...)
    ↓
المستند المحمل حاليًا داخل WebView
```

`evaluateJavascript()` API شرعية بحد ذاتها.

الخطر يعتمد على provenance:

```text
من يتحكم في السكربت؟
من يستطيع تعديله؟
أي صفحة ستنفذه؟
أي Native Bridge مكشوف وقت التنفيذ؟
هل توجد جلسة مستخدم مصادق عليها؟
```

عندما يجتمع تنفيذ script وقت التشغيل مع Native Bridge غني بالقدرات، تصبح ضوابط origin وprovenance حاسمة.

في لقطة المصدر المحللة، تُستهلك بيانات script وقت التشغيل داخل مسار الحقن، لكن شجرة المصدر لا تثبت أين تُملأ هذه القيمة أثناء التشغيل. لذلك يؤكد البحث **وجود تنفيذ JavaScript وقت التشغيل**، بينما يترك **مصدر هذا السكربت** غير مثبت دون أدلة runtime إضافية.

---

# 5. رصد إدخال DOM (DOM Input Instrumentation)

تراقب JavaScript المحللة عناصر تفاعلية شائعة مثل:

```text
input
textarea
button
a
select
```

وأحداثًا تشمل:

```text
focus / focusin
input
keydown
keyup
blur
click
resize
```

تمثيل مخفض عمدًا:

```javascript
document.addEventListener("input", event => {
    if (event.target.matches("input, textarea")) {
        // ملاحظة حالة الحقل.
    }
});
```

يمكن للتنفيذ المحلل تمرير معلومات مختارة من الحقول إلى طبقة Native عبر JavaScript Bridge.

مسار الثقة يصبح:

```text
إدخال المستخدم
    ↓
حدث DOM
    ↓
JavaScript محقونة
    ↓
Native Bridge
    ↓
معالجة / تخزين / إرسال داخل Native
```

---

# 6. مسارات الصلاحيات الأصلية

يشير المنطق الأصلي المرتبط بالعينة إلى فئات قدرات حساسة مثل:

```text
Camera
Location
Microphone
Call Log
Contacts
SMS
Phone
File Access
Overlay
All Files Access
Accessibility
Battery Optimization
```

هذا **لا يعني** أن JavaScript تمنح صلاحيات Android مباشرة. يظل Android مسؤولًا عن permission dialogs وواجهات Settings المحمية. كما تعتمد إمكانية المنح على Manifest وإصدار Android وtarget SDK ودور التطبيق وقواعد special access وسياسات التوزيع.

المشكلة الأمنية هي:

> **قد يستطيع تنفيذ يتحكم به الويب بدء أو تنسيق مسارات Native مرتبطة بقدرات Android حساسة.**

وهذا قد يحول WebView من عارض محتوى إلى **سطح تنسيق للصلاحيات**: المستخدم يثق في تجربة الويب، بينما يستخدم التطبيق هذا السياق لدفعه نحو قدرات Native حساسة.

---

## شروط الاستغلال وإمكانية الوصول

وجود هذه البنية لا يعني أن كل موقع محمل يتم استغلاله تلقائيًا.

يتطلب مسار إساءة استخدام عملي واحدًا أو أكثر من الآتي:

```text
صفحة يسيطر عليها مهاجم
Origin موثوق مخترق
Redirect / Navigation غير آمن
Script أو frame من طرف ثالث قادر على تنفيذ JavaScript
مصدر runtime script مخترق
منطق حقن داخل التطبيق يستهدف الصفحة المحملة
```

وبالتزامن:

```text
يبقى Native Bridge مكشوفًا
        +
تنتج إحدى دوال Native القابلة للاستدعاء أثرًا أمنيًا ذا قيمة
```

بالنسبة للصلاحيات، قد يظل Android بحاجة إلى تفاعل صريح من المستخدم أو المرور عبر Settings محمية. لذلك تتعلق النتيجة **بمن يستطيع بدء وصياغة رحلة الصلاحية**، لا بمنح كل الصلاحيات بصمت.

كذلك، فإن إعلانًا يعمل في **WebView منفصل يملكه Advertising SDK** لا يرث تلقائيًا bridge مسجلًا في WebView آخر. الخطر هنا يخص محتوى طرف ثالث ينفذ داخل نفس WebView المزود بالجسر أو يصل إلى نفس الواجهة المكشوفة.

---

# 7. الإطارات التابعة للأطراف الثالثة ومحتوى الإعلانات

تحذر إرشادات Android من أن `addJavascriptInterface()` يحقن كائن Java في **كل frame داخل WebView، بما في ذلك iframes**.

لذلك:

```text
الصفحة المضيفة
   │
   ├── First-party script
   │
   └── Third-party iframe / embedded content
                     ↓
              Native Bridge
```

ما زالت Same-Origin Policy تحمي أنواعًا كثيرة من تفاعل DOM بين origins مختلفة، لكن:

> **عزل DOM بين origins ليس هو نفسه عزل Native Bridge.**

وهذه نقطة حاسمة عندما تتضمن الصفحة إعلانات أو analytics أو widgets أو مكونات دفع مضمنة أو frames خارجية.

المراجع:

- Android Developers — https://developer.android.com/privacy-and-security/risks/insecure-webview-native-bridges
- Android Developers — https://developer.android.com/develop/ui/views/layout/webapps/webview

---

# 8. سياق الإعلانات والتوزيع عبر الأطراف الثالثة

## ما تثبته العينة

العينة لا تثبت أنها وُزعت عبر AdMob أو Meta Ads أو Google Ads أو TikTok Ads أو أي شبكة إعلان.

لذلك لا ينبغي نسبة قناة توزيع إعلانية للعينة من هذا المصدر وحده.

## الخطر المعماري

إذا تم عرض HTML/JavaScript تابع لإعلان داخل **نفس WebView المزود بالجسر**، فإنه يدخل إلى بيئة ثقة حساسة:

```text
التطبيق
    ↓
Bridge-enabled WebView
    ↓
صفحة تتضمن إعلانًا / iframe من طرف ثالث
    ↓
Third-party JavaScript
    ↓
Native bridge exposure
```

إمكانية استغلال إعلان محدد للجسر تعتمد على نموذج frames الفعلي، ومدى كشف bridge، وقواعد التنقل، وإصدار Android، وكود التطبيق.

## الارتباط بالعالم الحقيقي

تؤكد استخبارات التهديدات العامة أن malvertising يُستخدم فعلًا لتوزيع Android financial malware:

- **2024 — PWA/WebAPK banking phishing:** وثقت ESET إعلانات شبكات اجتماعية تقود الضحايا إلى صفحات مصرفية / Google Play مزيفة ثم إلى PWA أو WebAPK ضارة.
- **2025 — Crocodilus:** وثقت ThreatFabric إعلانات Facebook توصل dropper لتروجان مصرفي Android ثم توسعًا جغرافيًا لاحقًا.
- **2026 — StreamRat:** وثقت ThreatFabric إعلانات Meta وTikTok تنتحل خدمة بث تلفزيوني مجانية، ووصلت الحملة إلى ما يقدر بنحو 570 ألف ضحية محتملة قبل تنزيل Android banking RAT يستخدم Accessibility وoverlays وkeylogging والتحكم عن بعد.

هذه الحملات لا تثبت أن كل إعلان يصل إلى WebView bridge. لكنها تثبت أن الإعلان أصبح قناة واسعة لاكتساب الضحايا وبناء مسار الثقة والتركيب الذي تحتاجه عمليات الاحتيال الحديثة.

المراجع:

- https://www.welivesecurity.com/en/eset-research/be-careful-what-you-pwish-for-phishing-in-pwa-applications/
- https://www.threatfabric.com/blogs/crocodilus-mobile-malware-evolving-fast-going-global
- https://www.threatfabric.com/blogs/from-meta-ads-to-full-device-takeover-uncovering-streamrat

---

# 9. المواقع المصرفية الشرعية داخل WebView خبيث

نعم — **على مستوى العميل/WebView**.

لا — ليس تلقائيًا على خادم البنك.

يمكن للبنك أن يملك:

```text
HTTPS صحيح
شهادة TLS صحيحة
خوادم غير مخترقة
نطاق صحيح
Backend آمن
```

ومع ذلك تُعرض صفحته داخل تطبيق Android غير موثوق.

النموذج:

```text
خادم البنك الشرعي
        ↓ HTTPS
WebView خبيث / غير آمن داخل Android
        ↓
JavaScript محقونة محليًا
        ↓
مراقبة DOM / Cookie / Session
        ↓
التطبيق الأصلي
```

يحمي TLS الاتصال أثناء النقل، لكنه لا يمنع التطبيق الذي يملك WebView من رصد سياق العرض أو تعديله داخل بيئته.

---

# 10. الارتباط المباشر ببرمجيات الاحتيال المصرفي

## 10.1 Brokewell — 2024

وثقت ThreatFabric أن Brokewell يحمّل **الموقع الشرعي** داخل WebView الخاص به.

ويتجاوز `onPageFinished()`، وينتظر نشاط تسجيل الدخول، ثم يحصل على session cookies ويرسلها إلى بنية القيادة والتحكم.

وهذا يثبت المفهوم الاستراتيجي نفسه:

> **لا يحتاج الموقع الشرعي لأن يكون مخترقًا؛ التطبيق العدائي يسيطر على سياق WebView على جهة العميل.**

كما وثقت ThreatFabric توسعًا سريعًا للقدرات، وAccessibility logging، وoverlays، وتحكمًا عن بعد، وخطرًا على عملاء المؤسسات المالية.

المرجع:

- https://www.threatfabric.com/blogs/brokewell-do-not-go-broke-by-new-banking-malware

## 10.2 Sturnus — 2025

وثقت ThreatFabric تروجانًا مصرفيًا يطلق محرك HTML overlay داخل WebView مهيأ بـ:

```text
JavaScript
DOM storage
JavaScript bridge
```

ويعترض bridge البيانات التي يدخلها الضحية ويرسلها إلى C2.

كما يجمع Sturnus ذلك مع Accessibility-based keylogging والتحكم عن بعد.

وهذا analogue عام قريب من:

```text
WebView
→ JavaScript
→ Native Bridge
→ Data Exfiltration
```

المرجع:

- https://www.threatfabric.com/blogs/sturnus-banking-trojan-bypassing-whatsapp-telegram-and-signal

## 10.3 Crocodilus — 2025

يجمع Crocodilus بين:

```text
Accessibility abuse
Overlay phishing
Credential interception
Remote control
Hidden / black-screen operation
```

ثم وثقت ThreatFabric لاحقًا توسعه عالميًا واستخدام الإعلانات الاجتماعية الخبيثة.

المراجع:

- https://www.threatfabric.com/blogs/exposing-crocodilus-new-device-takeover-malware-targeting-android-devices
- https://www.threatfabric.com/blogs/crocodilus-mobile-malware-evolving-fast-going-global

## 10.4 PlayPraetor — 2025

وثقت Cleafy عملية Android RAT / banking fraud واسعة أصابت أكثر من 11 ألف جهاز خلال أقل من ثلاثة أشهر.

واستهدفت ما يقارب 200 تطبيق مصرفي ومحفظة Crypto موزعة عالميًا عبر بنية MaaS متعددة المستأجرين.

المرجع:

- https://www.cleafy.com/cleafy-labs/playpraetors-evolving-threat-how-chinese-speaking-actors-globally-scale-an-android-rat

## 10.5 TrickMo — 2026

وثقت ThreatFabric توزيعًا نشطًا لـTrickMo يستهدف تطبيقات بنكية وFintech ومحافظ وتطبيقات Authenticator.

ومن القدرات:

```text
Fullscreen WebView credential overlays
Keylogging
Accessibility-assisted device control
SMS / notification interception
Screen streaming
On-device network pivoting
```

وأظهرت campaign tags نشاطًا متوازيًا يطال عملاء في فرنسا وإيطاليا والنمسا.

المرجع:

- https://www.threatfabric.com/blogs/new-trickmo-variant-device-take-over-malware-targeting-banking-fintech-wallet-auth-app

## 10.6 StreamRat — 2026

وثقت ThreatFabric حملة واسعة مدفوعة بالإعلانات عبر Meta وTikTok.

ويجمع malware بين:

```text
Credential overlays
UI-tree collection
Keylogging
Accessibility abuse
VNC / remote control
Hidden-screen operation
```

وهو يوضح تقارب:

```text
إعلانات واسعة
        +
هندسة اجتماعية
        +
اكتساب صلاحيات
        +
Device Takeover
        +
سرقة بيانات مالية
```

المرجع:

- https://www.threatfabric.com/blogs/from-meta-ads-to-full-device-takeover-uncovering-streamrat


---

# 11. السلالة التاريخية: من Remote C2 إلى منصات الاحتيال الديناميكية في Android

يشير السجل التاريخي إلى أن التحول نحو **malware ديناميكي في Android** بدأ قبل SpyNote وSpyMax بسنوات.

في هذا البحث، لا تعني كلمة "ديناميكي" فقط أن الجهاز يمكن التحكم به عن بُعد، بل تصف تطورًا انتقل فيه جزء متزايد من السلوك الهجومي **إلى خارج APK الثابت**: أوامر بعيدة، قدرات يتم تنزيلها لاحقًا، scripts قابلة للتحديث، محتوى ويب ديناميكي، ثم لاحقًا طبقات قرار تكيفية.

نموذج تاريخي مفيد:

```text
APK خبيث ثابت
        ↓
تنفيذ أوامر بعيدة
        ↓
قدرات قابلة للتنزيل / الاستبدال
        ↓
Scripts / سلوك قابل للتحديث
        ↓
لوحات تحكم RAT تجارية
        ↓
Overlay + Accessibility + WebView abuse
        ↓
Web-to-Native fraud workflows ديناميكية
        ↓
Device Takeover / On-Device Fraud
        ↓
احتمال تنسيق تكيفي مدعوم بالذكاء الاصطناعي
```

التواريخ أدناه تمثل **أقدم دليل علني موثق ظهر ضمن المصادر التي شملتها هذه المراجعة**، وليس إثباتًا بأن تطبيقًا أقدم أو خاصًا لم يوجد.

## 11.1 عام 2010 — Geinimi وانتقال Android نحو نموذج botnet

في ديسمبر **2010**، وصفت Lookout برمجية **Geinimi** بأنها أول malware على Android تمت ملاحظته في البرية بقدرات شبيهة بالـbotnet.

التحليل العام وثق قدرتها على الاتصال ببنية بعيدة ووجود مسارات يمكنها تلقي أوامر من خادم.

ومن القدرات المذكورة:

```text
جمع معلومات الجهاز / الموقع
الاتصال بخادم بعيد
تنزيل تطبيقات إضافية
بدء مسارات تثبيت تطبيقات
إمكانية تلقي أوامر عن بعد
```

أهمية هذه المرحلة معمارية؛ فهي تمثل انتقالًا مبكرًا من:

```text
تطبيق خبيث = سلوك محلي ثابت
```

إلى:

```text
تطبيق خبيث
    ↓
بنية بعيدة
    ↓
سلوك يختاره المهاجم
```

وتضع مجموعات بيانات أكاديمية لاحقة Geinimi في عام **2010** ضمن بدايات نشاط Android botnets الموثق.

المراجع:

- SecurityWeek — Sophisticated New Android Trojan “Geinimi” Spreading in China  
  https://www.securityweek.com/sophisticated-new-android-trojan-geinimi-spreading-china/
- University of New Brunswick — Android Botnet Dataset  
  https://www.unb.ca/cic/datasets/android-botnet.html

### التفسير التاريخي

لا ينبغي وصف Geinimi كـRAT حديث مكافئ لـSpyNote أو SpyMax.

قيمتها في السلسلة التاريخية هي:

> **البنية البعيدة أصبحت جزءًا من سلوك malware في Android منذ 2010 على الأقل.**

وهذا يسبق الظهور العلني لـSpyNote بنحو ست سنوات، والتاريخ الموثق لتطوير SpyMax بنحو تسع سنوات.

---

## 11.2 عام 2011 — القدرة القابلة للتنزيل تصبح تشغيلية

يمثل جيل **DroidDream** عام 2011 خطوة مهمة أخرى.

وثقت تقارير DroidDream وDroidDream Light قدرات اتصال بـC2 وتنزيل برمجيات إضافية بعد التثبيت.

كما استخدمت بعض نسخ DroidDream الأصلية privilege escalation لتثبيت كود إضافي بتدخل أقل من المستخدم على أجهزة ضعيفة، بينما احتفظت نسخ أخرى بقدرة التنزيل مع احتمال الحاجة لموافقة المستخدم.

الأهمية المعمارية:

```text
التطبيق الأولي
        ↓
البنية البعيدة
        ↓
قدرة إضافية
```

من لحظة إمكانية إضافة قدرات بعد التثبيت، لا يعود APK بالضرورة هو كامل الهجوم.

هذا المبدأ أصبح لاحقًا أساسيًا في RATs المعيارية ومنظومات banking malware.

المرجع:

- Ars Technica — DroidDream Light  
  https://arstechnica.com/gadgets/2011/06/droiddream-light-a-malware-nightmare-booted-from-android-market-encumbered-boobies-out-of-android-marketdroiddream-light-an-malware-nightware-booted-from-android-market/

---

## 11.3 عام 2012 — Luckycat يثبت تطوير Android RAT داخل بنية APT

يمثل تحقيق **Luckycat** دليلًا مبكرًا مهمًا في هذه السلالة.

في **2012**، عثرت Trend Micro على تطبيقين Android داخل بنية مرتبطة بحملة Luckycat.

كانت التطبيقات في مرحلة تطوير / proof-of-concept، لكنها قادرة على:

```text
استقبال أوامر من C2 بعيد
تنفيذ أعمال يحددها المهاجم
جمع معلومات الجهاز
استعراض المجلدات
رفع الملفات
تنزيل الملفات
دعم remote shell كان ما يزال قيد التطوير
```

ووصفت Trend Micro هذه التطبيقات بأنها شبيهة بـRemote Access Trojans.

أهمية ذلك كبيرة: بحلول 2012 كان هناك تطوير هجومي مستهدف مبني حول **remote-command architecture** بدل payload ثابت.

المرجع:

- Trend Micro — Adding Android and Mac OS X Malware to the APT Toolbox  
  https://documents.trendmicro.com/assets/wp/wp_adding-android-and-mac-osx-malware-to-the-apt-toolbox.pdf

### لماذا 2012 سنة مهمة؟

تعامل هذه الدراسة **2012 كسنة تحول مهمة**.

فالسجل العام يدعم عندها معمارية تتضمن:

```text
C2 بعيد ومستمر
        +
أوامر يحددها المهاجم
        +
رفع / تنزيل ملفات
        +
remote shell مخطط له
```

وهذا أقرب بكثير إلى RATs اللاحقة من malware بسيط لسرقة بيانات أو premium SMS.

---

## 11.4 أواخر 2012–2013 — AndroRAT يحول التحكم عن بعد إلى سلعة

ظهر **AndroRAT** في منتديات سرية أواخر **2012** وأصبح متداولًا علنًا في 2013.

قدم لوحة تحكم عن بعد سهلة نسبيًا وسمح بأعمال مثل:

```text
مراقبة / بدء المكالمات
مراقبة / إرسال SMS
الحصول على GPS
تشغيل الكاميرا / الميكروفون
الوصول إلى ملفات الجهاز
```

وبحلول منتصف 2013، بدأت APK binders تُباع لتسهيل دمج AndroRAT في تطبيقات Android شرعية.

وهنا انتقل النظام البيئي من:

```text
تطوير malware متخصص
```

إلى:

```text
RAT قابل لإعادة الاستخدام
    +
لوحة تحكم
    +
Binding / Repackaging شبه آلي
    +
خفض متطلبات مهارة المشغل
```

وهذه بداية نمط سيصبح لاحقًا محوريًا في crimeware-as-a-service.

المرجع:

- SecurityWeek — Hackers Sell APK Binders for Google Android Remote Access Tool  
  https://www.securityweek.com/hackers-sell-apk-binders-google-android-remote-access-tool/

---

## 11.5 عام 2013 — scripts قابلة للتحديث تجعل النموذج الديناميكي صريحًا

وثقت Trend Micro عائلة **ANDROIDOS_KSAPP** عام 2013 كـAndroid backdoor قادرة على استقبال وتنفيذ أوامر من مستخدم بعيد.

الأهم لهذه الدراسة أنها وثقت قدرة malware على **تحديث script تلقائيًا لتجنب الاكتشاف**.

وهذا يمثل قفزة مفاهيمية:

```text
أمر بعيد
        ↓
تعديل السلوك عن بعد
        ↓
تحديث script
```

لم يعد السلوك الخبيث محددًا بالكامل بما كان موجودًا عند التثبيت.

وهذا مثال مبكر علني على المبدأ:

> **أبقِ العميل المثبت ثابتًا قدر الإمكان، وغيّر السلوك من الخارج.**

المرجع:

- Trend Micro — Malicious and High-Risk Android Apps Hit 1 Million: Where Do We Go from Here?  
  https://www.trendmicro.com/vinfo/us/security/news/mobile-safety/malicious-and-high-risk-android-apps-hit-1-million-where-do-we-go-from-here

---

## 11.6 2013–2014 — SandroRAT / DroidJack يرسخان النموذج التجاري

سلالة **SandroRAT → DroidJack** توضح انتقالًا آخر: التجارة والتنميط.

السجل العام يربط:

```text
2013 — Sandroid / SandroRAT lineage
        ↓
2014 — DroidJack
```

تم بيع DroidJack تجاريًا وقدم قدرات واسعة دون الحاجة إلى root.

ومن الوظائف الموثقة:

```text
تثبيت APKs
نسخ الملفات
قراءة الرسائل
مراقبة المكالمات
الوصول إلى جهات الاتصال
تسجيل الميكروفون / الكاميرا
الحصول على الموقع
تنفيذ أوامر بعيدة
```

يمثل هذا نضج RAT في Android كمنتج قابل للتوزيع والبيع، وليس مجرد عينة malware.

المرجع:

- SecurityWeek — Developers of Android RAT DroidJack Traced to India  
  https://www.securityweek.com/developers-android-rat-droidjack-traced-india/

---

## 11.7 2015–2016 — Banking overlays وSpyNote يلتقيان مع التحكم عن بعد

بحلول **2015**، وثقت CERT Polska برمجية **GMBot** المصرفية واستخدامها overlays لتطبيقات Android، وقارنت التقنية مباشرة بـwebinjects في احتيال البنوك.

وبحلول **2016**، وثقت Mandiant حملات أوروبية تستخدم Android view overlays لتقليد تطبيقات شرعية وسرقة بيانات الدخول.

في الفترة نفسها تقريبًا، أصبح **SpyNote** ظاهرًا علنًا.

وذكرت التقارير في 2016 builder/tooling مسربة لـSpyNote مع قدرات مثل:

```text
SMS
microphone
camera
contacts
location
device data
remote-control functions
```

أهمية هذه المرحلة أن خطين كانا يتطوران بدأا يلتقيان داخل النظام البيئي:

```text
Remote Administration
        +
Credential / Interface Deception
```

المراجع:

- CERT Polska — GMBot  
  https://cert.pl/posts/2015/10/lang_plgmbot-androidowa-ubozsza-wersja-webinjectowlang_pllang_engmbot-android-poor-mans-webinjectslang_en-2/
- Mandiant / Google Cloud — Android Overlay Malware  
  https://cloud.google.com/blog/topics/threat-intelligence/latest-android-overlay-malware-spreading-in-europe/
- Palo Alto Networks — SpyNote reported in 2016  
  https://www.paloaltonetworks.com/blog/2016/07/palo-alto-networks-news-of-the-week-july-30-2016/
- MITRE ATT&CK — SpyNote RAT  
  https://attack.mitre.org/software/S0305/

---

## 11.8 SpyNote وSpyMax — نقاط مهمة في السلالة وليستا أصل التحكم الديناميكي

تدعم الأدلة المتاحة التسلسل التالي:

```text
SpyNote ظاهر علنًا بحلول 2016
        ↓
SpyMax موثق كتطوير نحو 2019
        ↓
تسريب/نشر كود SpyMax نحو 2020
```

تسمية عائلات malware في هذا المجال غير متسقة تاريخيًا.

بعض المصادر تستخدم SpyNote وSpyMax كأسماء متداخلة أو مرتبطة، بينما تميز مصادر أخرى بين فروع تطوير مختلفة.

لذلك لا تقول هذه الورقة:

```text
SpyNote أنشأ SpyMax
SpyMax أنشأ SpyNote
كل عينة SpyNote == كل عينة SpyMax
```

تدعم الأدلة العامة الاستنتاج التالي:

> **SpyNote يسبق التاريخ العلني الموثق لتطوير SpyMax، بينما أصبح SpyMax لاحقًا نقطة consolidation وانتقال كود مهمة في منظومة Android RATs المعيارية.**

المراجع:

- Group-IB — From SpyMax to Craxs RAT  
  https://www.group-ib.com/blog/craxs-rat-malware/
- UNODC — Developments in Cyber-Enabled Fraud and Technological Innovation (2024)  
  https://www.unodc.org/roseap/uploads/documents/Publications/2024/TOC_Convergence_Report_2024.pdf
- ERNW / Insinuator — SpyMax Technical Analysis  
  https://insinuator.net/2022/09/spymax-the-android-rat-and-it-works-like-that/

---

## 11.9 لماذا يبقى SpyMax مهمًا تاريخيًا؟

تذكر Group-IB أن SpyMax بُني تقريبًا في **2019** وأن كوده أصبح منشورًا/مسرّبًا في **2020**، ثم أعيد استخدامه وتخصيصه من جهات أخرى.

كما توثق تحليلات تقنية مستقلة قدرات مثل:

```text
remote control
broad Android permissions
Accessibility-based keylogging
camera / microphone access
location
SMS / contacts / files
dynamic loading of additional components
remote commands
```

أهمية SpyMax هنا ليست الاختراع، بل **التجميع والانتشار**.

وجود معمارية قابلة لإعادة الاستخدام مع كود مسرب يمكن أن يسرع:

```text
forking
customization
feature transfer
operator adoption
derivative malware families
```

وهذا يفسر أهميته داخل السلالة التي أدت إلى مشتقات لاحقة.

---

## 11.10 عام 2021 — S.O.V.A. يثبت إساءة استخدام الموقع الشرعي والجلسة داخل WebView

يصبح السجل العام مرتبطًا مباشرة بفرضية هذه الورقة في **2021**.

وثقت ThreatFabric أن **S.O.V.A.** يفتح URL شرعيًا داخل WebView يملكه malware ويستخدم Android `CookieManager` للحصول على session cookies بعد تسجيل الدخول.

النموذج:

```text
موقع شرعي
        ↓
WebView يملكه malware
        ↓
جلسة مصادق عليها
        ↓
استخراج Session Cookie
```

كما استخدم S.O.V.A. Accessibility Services وoverlays وkeylogging وWebView injections.

المرجع:

- ThreatFabric — S.O.V.A.  
  https://www.threatfabric.com/blogs/sova-new-trojan-with-fowl-intentions

وهو أحد أوضح السوابق العامة لنموذج رصد الموقع الشرعي الذي يناقشه هذا البحث.

---

## 11.11 عام 2022 — Xenomorph يجعل WebView + JavaScript + Native Bridge صريحًا في banking malware

في **2022**، وثقت ThreatFabric أن **Xenomorph** ينشئ banking overlay WebViews مع JavaScript مفعلة وواجهة Native مسجلة عبر `addJavascriptInterface(...)`.

كما استخدم Accessibility Services ومحتوى overlay يتم جلبه ديناميكيًا.

وهذا يثبت أن نظام banking malware في Android كان يحتوي بحلول 2022 على التركيب:

```text
WebView
    +
JavaScript
    +
Native Bridge
    +
Dynamic Overlay Content
    +
Accessibility
```

المرجع:

- ThreatFabric — Xenomorph  
  https://www.threatfabric.com/blogs/xenomorph-a-newly-hatched-banking-trojan

وهذا أقرب بكثير إلى البنية متعددة الطبقات المحللة في هذه الدراسة من أجيال RAT المبكرة.

---

## 11.12 2023–2024 — الأدوات المسربة والمكسورة تتحول إلى hostile supply chain

طورت منظومة Android RAT أيضًا مشكلة في supply chain الخاصة بالأدوات.

ذكرت CYFIRMA في أغسطس 2023 أن بعض نسخ CraxsRAT المكركة كانت backdoored، وأن builders مكركة على Windows ظهرت مع malware أو ransomware مضاف مسبقًا.

كما ذكرت Broadcom أن CraxsRAT كُسر وسُرّب مجانًا، ما وسع الوصول إليه.

ينتج عن ذلك نظام ثانٍ:

```text
Private / Commercial RAT
        ↓
Cracking / Reverse Engineering
        ↓
Leaked Builders / Code
        ↓
Backdoored Cracked Tooling
        ↓
Threat Actors Become Targets
        ↓
Credential / Source / Infrastructure Exposure
```

يمكن لهذا أن يسرع انتشار الكود وعمليات counter-compromise داخل النظام الإجرامي.

تدعم الأدلة المذكورة وجود tooling مكرك وbackdoored واتساع توزيع الكود، لكنها لا تثبت إسنادًا محددًا للجهات المسؤولة عن أي compromises لاحقة.

المراجع:

- CYFIRMA — Unmasking EVLF DEV / CraxsRAT  
  https://www.cyfirma.com/research/unmasking-evlf-dev-the-creator-of-cypherrat-and-craxsrat/
- Broadcom — CraxsRAT  
  https://www.broadcom.com/support/security-center/protection-bulletin/craxsrat
- Group-IB — From SpyMax to Craxs RAT  
  https://www.group-ib.com/blog/craxs-rat-malware/

---

## 11.13 2024–2026 — التقارب، MaaS، malvertising، والتوسع الدولي

تشير الأدلة المراجعة إلى أن **2024–2026 تمثل مرحلة تسارع وتقارب** وليست أصل التقنيات.

### 2024 — إساءة استخدام المواقع الشرعية وتحسين التشغيل

وثقت التقارير:

```text
Brokewell legitimate-site WebView/session abuse
تطوير سريع لقدرات Device Takeover
PWA/WebAPK banking phishing عبر social media
Medusa On-Device Fraud
ToxicPanda Account Takeover / On-Device Fraud
```

**التقييم:** 2024 يظهر تركيبات ناضجة من trusted web interaction وsession abuse وoverlays والتحكم عن بعد والقدرات الأصلية.

### 2025 — التخصص ومنصات الاحتيال القابلة للتوسع

وثقت التقارير:

```text
Crocodilus geographic expansion
social-media malvertising
PlayPraetor MaaS scaling
Sturnus WebView JavaScript bridges
Accessibility logging
hidden remote control
operator infrastructure specialization
```

**التقييم:** 2025 يمثل انتقالًا من تقنيات منفردة إلى منصات احتيال متكاملة.

### 2026 — acquisition دولي وDevice Takeover

وثقت التقارير:

```text
TrickMo banking / wallet campaigns متوازية
StreamRat Meta / TikTok acquisition
credential overlays
keylogging
Accessibility abuse
screen streaming
hidden-screen operation
Device Takeover
```

**التقييم:** بحلول 2026، أمكن تركيب مكونات تطورت خلال عقد كامل داخل عمليات احتيال دولية صناعية.

---

## 11.14 النموذج التاريخي

يصبح التسلسل:

```text
2010
Geinimi
remote C2 / botnet-like Android control
        ↓
2011
DroidDream
downloadable post-install capability
        ↓
2012
Luckycat Android RAT
targeted C2 + upload/download + remote-shell development
        ↓
late 2012–2013
AndroRAT
commodity remote-control panel + binders
        ↓
2013
KSAPP
remote commands + automatic script updates
        ↓
2013–2014
SandroRAT / DroidJack
commercial RAT maturation
        ↓
2015–2016
banking overlays + SpyNote
        ↓
2019–2020
SpyMax consolidation + implementation leak
        ↓
2021
S.O.V.A.
legitimate-site WebView/session abuse
        ↓
2022
Xenomorph
WebView + JavaScript + native bridge + dynamic overlays
        ↓
2023–2024
cracked / leaked RAT supply-chain expansion
        ↓
2024–2026
MaaS + malvertising + ODF + Device Takeover
        ↓
نموذج التهديد التالي
AI-adaptive trust manipulation
```

---

## 11.15 ما الذي يثبته الخط الزمني — وما الذي لا يثبته؟

تدعم الأدلة تطورًا معماريًا طويلًا.

لكنها **لا تثبت** أن مطورًا واحدًا أو متحكمًا مخفيًا صمم كامل المسار.

هناك تفسير طبيعي بديل:

```text
تقنية ناجحة
        ↓
نسخ / إعادة استخدام
        ↓
تسريب أدوات
        ↓
تجارة
        ↓
منافسة
        ↓
Feature convergence
```

كما قد تكون أدوات خاصة موجودة قبل أول اكتشاف عام لها.

لذلك اللغة التاريخية الصحيحة هي:

> **earliest publicly documented evidence**

وليست:

> **first implementation ever**

كذلك، لا يثبت الظهور المبكر لهذه المبادئ أن المطورين في 2012 توقعوا بدقة شكل هجمات 2026.

ما يثبته هو أن مبادئ أساسية في malware التكيفي الحديث فُهمت مبكرًا جدًا:

```text
اجعل التحكم خارج APK قدر الإمكان
افصل القدرات عن التثبيت الأولي
غيّر السلوك بعد النشر
اخفض مهارة المشغل عبر tooling قابل لإعادة الاستخدام
استغل واجهات موثوقة
انقل القرار نحو البنية البعيدة
```

ثم أصبحت هذه المبادئ لاحقًا أساسًا لمنظومات احتيال Android الحديثة.

---

## 11.16 الخلاصة التاريخية

> **يسبق malware الديناميكي في Android كلًا من SpyNote وSpyMax بسنوات. يدعم السجل العام وجود remote-command / botnet-like control بحلول 2010، وقدرات يتم تنزيلها بعد التثبيت بحلول 2011، وتطوير Android RAT مستهدف مع C2 ورفع/تنزيل وremote shell بحلول 2012، وscripts خبيثة قابلة للتحديث تلقائيًا بحلول 2013. ثم ساهم AndroRAT وSandroRAT وDroidJack في تحويل التحكم عن بعد إلى tooling قابل للتجارة. ظهر SpyNote علنًا بحلول 2016، بينما جاء SpyMax لاحقًا كنقطة consolidation وانتقال كود مهمة وليس كأصل التحكم الديناميكي. وبحلول 2021–2022 ظهرت علنًا إساءة استخدام الجلسة داخل WebView وJavaScript/native bridges في banking malware، فيما تمثل 2024–2026 مرحلة تقارب وتجارة واكتساب عبر malvertising وتوسع دولي، لا مرحلة اختراع.**

يتتبع هذا الخط الزمني **التحول المعماري نفسه** بدل البدء بأسماء منتجات لاحقة مثل SpyNote أو SpyMax.

---

# 12. تقييم الأثر: Trust Conversion واكتساب الصلاحيات

لا يوجد benchmark عالمي يصنف جميع تقنيات Android الهجومية بمقياس واحد للأثر أو الفاعلية. لذلك يركز التقييم على البنية التقنية والأدلة المتاحة.

يشير التقييم البحثي إلى:

> **يمكن لنمط WebView-to-Native متعدد الطبقات أن يدعم مسارات عالية الأثر لاكتساب الصلاحيات وDevice Takeover في malware المالي الحديث. ويستخدم هذا البحث نموذج “Trust Conversion” لشرح هذا التدرج؛ ولا تثبت الأدلة المراجعة ترتيبًا عالميًا موحدًا بين تقنيات الهجوم على Android.**

الميزة لا تأتي من bypass واحد، بل من جمع عدة أنواع من الثقة:

```text
Trusted Brand
    +
Legitimate or convincing Web Content
    +
User Interaction
    +
Android Native UI
    +
Permission Workflow
    +
Accessibility / Overlay / Remote Control
```

قد يعتقد المستخدم أنه يتعامل مع:

```text
بنكه
مزود دفع
متصفحه
تحديث نظام
فحص أمني
تطبيق موثوق
```

بينما يتحول هذا السياق تدريجيًا إلى قدرات Native وصلاحيات يوافق عليها المستخدم.

وتحذر Google من أن منح Accessibility لتطبيق ضار قد يسمح بالتحكم بالجهاز والوصول إلى معلومات حساسة مثل البيانات المصرفية.

المرجع:

- https://blog.google/security/whats-new-in-android-security-privacy-2025/

---

# 13. نموذج Trust Conversion

**Trust Conversion** نموذج تحليلي تستخدمه هذه الورقة، وليس taxonomy رسمية في Android أو OWASP أو MITRE ATT&CK أو الأدبيات الأكاديمية.

الفكرة ليست “سرقة الصلاحية” مباشرة.

لا يزال Android يتطلب من المستخدم أو النظام تفويض قدرات محمية كثيرة.

المهاجم يحاول بدلًا من ذلك **تحويل الثقة**:

```text
Brand Trust
    ↓
Web Trust
    ↓
Interaction Trust
    ↓
Native UI Trust
    ↓
Permission Approval
    ↓
Device Capability
```

الميزة تقنية ونفسية في الوقت نفسه.

طلب صلاحية يظهر فجأة قد يكون مريبًا.

لكن الطلب الذي يبدو جزءًا من رحلة بنكية أو استرداد حساب أو فحص احتيال أو تحديث أو تحقق قد يكون أكثر إقناعًا.

وهذا يساعد في تفسير استمرار جاذبية Accessibility وOverlay وnotification access وscreen sharing لعمليات الاحتيال الحديثة.

---

# 14. ثقة العلامة التجارية كسطح هجوم ضد العملاء

لا تحتاج الشركة لأن يُخترق خادمها حتى تُستخدم ثقة العملاء بها ضدهم.

يمكن للمهاجم إعادة إنتاج أو استغلال:

```text
هوية العلامة
روابط شرعية
مسارات تسجيل الدخول
استرداد الحساب
لغة الدعم
تحذيرات الأمان
صور التطبيق
مسارات الدفع / التحقق
```

النموذج:

```text
Company Trust
      ↓
Hostile Client Environment
      ↓
Customer Interaction
      ↓
Credential / Session / Permission Capture
```

وبالتالي:

> **قد تبقى الشركة غير مخترقة تقنيًا، بينما تُستخدم علاقتها الموثوقة مع العميل كسلاح.**

رحلة العميل التي تفترض أن بيئة الجهاز موثوقة يمكن أن تتحول إلى سطح احتيال ضد العملاء.

وهذا لا يعني أن الشركة مسؤولة عن أي malware عشوائي على جهاز المستخدم.

المعنى هو أن:

```text
HTTPS
MFA
Backend آمن
Brand موثوق
```

لا تكفي وحدها عندما يكون الجهاز أو rendering environment نفسه عدائيًا.

---

# 15. الدلالات على المؤسسات المالية

صُممت عمليات **On-Device Fraud** الحديثة لجعل النشاط الاحتيالي يبدو صادرًا من جهاز الضحية الحقيقي.

وقد يضعف ذلك ضوابط تعتمد فقط على:

```text
Device identity
IP reputation
Known browser
Valid session
Correct password
Correct OTP
```

لأن malware قد يعمل بعد المصادقة أو داخل بيئة مستخدم سبق أن أصبحت موثوقة.

يشمل النموذج الدفاعي المقترح:

```text
Device integrity
Session integrity
Behavioral analysis
Transaction context
Risk-based authentication
Out-of-band confirmation
Trusted app / browser binding
Anti-overlay / anti-tamper controls
Runtime application protection
Fraud telemetry
```

ويشير بحث Brokewell من ThreatFabric تحديدًا إلى صعوبة Device Takeover أمام أنظمة مكافحة الاحتيال التي تعتمد بشكل كبير على device identification أو fingerprinting.


---

# 16. ما الذي تثبته العينة المحللة؟

تدعم العينة الاستنتاجات التالية:

- JavaScript مفعّلة داخل WebView.
- يوجد Native JavaScript Bridge مكشوف.
- يمكن تحميل محتوى شبكي ديناميكي.
- يوجد تنفيذ/حقن JavaScript وقت التشغيل.
- تتم مراقبة أحداث الإدخال والتفاعل داخل DOM.
- يمكن لبيانات قادمة من الويب العبور إلى Native.
- توجد مسارات Native للمعالجة / الإرسال.
- يمكن لتفاعل الويب التأثير على مسارات Native مرتبطة بالصلاحيات.
- تتم الإشارة إلى فئات قدرات Android حساسة.
- حد الثقة بين محتوى الويب ومنطق التطبيق الأصلي ضعيف.

---

# 17. ما الذي لا تثبته العينة؟

العينة وحدها **لا تثبت**:

```text
اختراق خادم بنك
اختراق backend لموقع شرعي
Remote Code Execution على خادم الموقع
تعديل دائم للموقع الشرعي
الانتشار الذاتي بين المواقع
الإصابة عبر كل إعلان
اختراق مستخدمين خارج التطبيق العدائي
منح صلاحيات Android تلقائيًا دون المستخدم/النظام
Universal Android sandbox escape
```

يجب أن تبقى هذه الحدود صريحة في أي Responsible Disclosure.

---

# 18. إرشادات الشدة والفرز

لا تعطي هذه الورقة CVSS عالميًا ولا product severity موحدة.

قد يتراوح النمط نفسه من design smell إلى ثغرة Critical حسب reachability والتأثير المثبت.

### أولوية المراجعة المعمارية

يجب التعامل مع النمط كحالة **عالية الأولوية للمراجعة الأمنية** عندما تتواجد:

```text
Untrusted / dynamic content
        +
Legacy bridge exposure
        +
Sensitive native methods
```

### الشدة الخاصة بالمنتج

قد يكون تصنيف **High** أو **Critical** مبررًا فقط عندما يثبت اختبار مصرح به مسارًا موثوقًا إلى نتائج مثل:

```text
Account takeover
Credential/session theft
Unauthorized financial transaction
High-impact data exfiltration
Sensitive permission acquisition
Device takeover
```

ويجب أخذ الآتي في الاعتبار:

```text
تفاعل المستخدم المطلوب
origin control
frame reachability
authentication state
إصدار Android
permission prerequisites
إمكانية إعادة إنتاج الأثر
```

لا ينبغي تصنيف الحالة Critical لمجرد وجود `addJavascriptInterface()`.

---

# 19. ملخص الإفصاح المسؤول

يكشف تطبيق Android دوال Native حساسة أمنيًا عبر WebView JavaScript Bridge، بينما يستطيع WebView نفسه تحميل محتوى شبكي ديناميكي وتشغيل JavaScript محقونة.

كما يرصد التنفيذ المحلل تفاعل DOM ويمكنه نقل بيانات مختارة قادمة من الويب إلى منطق التطبيق الأصلي. وتوجد مسارات Native مرتبطة بالصلاحيات ضمن حد الثقة نفسه.

توثق Android أن كائنات `addJavascriptInterface()` القديمة تُحقن في كل frame داخل WebView، بما في ذلك iframes، ما يجعل التحكم في origin ومحتوى الأطراف الثالثة حاسمًا.

إذا تم التنقل إلى موقع شرعي داخل WebView المتأثر، يستطيع التطبيق المضيف إضافة منطق رصد إلى الصفحة محليًا دون الحاجة إلى اختراق خادم الموقع.

يُصنَّف هذا النمط في البحث باعتباره **Web-to-Native trust-boundary failure / Cross-Layer WebView Trust Hijacking**، وليس اختراقًا مثبتًا لخادم الموقع.

وتوثق استخبارات التهديدات أمثلة تشغيلية قريبة في Android financial malware، منها إساءة استخدام الموقع الشرعي والجلسة في Brokewell، وجمع البيانات عبر JavaScript Bridge في Sturnus، وحملات لاحقة تجمع overlays وAccessibility abuse وmalvertising وDevice Takeover.

---

# 20. المعمارية الدفاعية

## 20.1 افصل WebViews الموثوقة عن غير الموثوقة

```text
Trusted internal WebView
      ↓
Minimal native capability

External / third-party content
      ↓
System browser أو isolated WebView
      ↓
No sensitive bridge
```

## 20.2 استخدم allowlist دقيقة للـOrigins

تحقق من:

```text
scheme
host
port
redirect destination
navigation state
```

ولا تستخدم قواعد واسعة مثل:

```text
https://*
```

## 20.3 أزل الجسر عندما تتغير حالة الثقة

```text
Trusted Origin
    ↓
Bridge enabled

Untrusted Origin
    ↓
Bridge unavailable
```

## 20.4 قلل وظائف Native المكشوفة

عندما تكون هناك حاجة إلى قناة Web-to-Native، توصي إرشادات Android الحديثة باستخدام messaging ذات قيود origin مثل:

```text
WebViewCompat.addWebMessageListener(...)
```

مع `allowedOriginRules` وsender-origin metadata، بدل التعامل مع `addJavascriptInterface()` القديم كما لو كان origin-aware.

لكن origin scoping هو **hardening وليس حل ثقة كاملًا**.

إذا نفذت JavaScript يسيطر عليها مهاجم داخل origin مسموح أصلًا، مثل XSS أو same-origin script compromise، فقد تستوفي قاعدة origin.

لذلك تبقى الحاجة إلى:

```text
Narrow capabilities
Payload / schema validation
Native authorization
Explicit user intent
Web-content integrity
```

وتوصي OWASP أيضًا بتقييد وظائف Native التي يكشفها WebView bridge.

النموذج:

```text
Narrow API
    ↓
Strict schema validation
    ↓
Native policy engine
    ↓
Allow / Deny
```

بدل command surfaces عامة.

## 20.5 لا تجعل محتوى الويب يقرر الصلاحيات

يجب أن تتطلب العمليات الحساسة:

```text
Explicit user intent
Trusted native UI
Current trusted origin
Valid application state
Feature-specific justification
Risk-policy approval
```

ولا ينبغي أن تكون نقرة DOM وحدها سلطة كافية لـAccessibility أو Overlay أو SMS أو microphone أو قدرات مشابهة.

## 20.6 عطّل قدرات WebView غير اللازمة

توصي Android بتعطيل وصول الملفات والمحتوى الخطير عند عدم الحاجة:

```java
setAllowUniversalAccessFromFileURLs(false);
setAllowFileAccess(false);
setAllowContentAccess(false);
```

واستخدام `WebViewAssetLoader` للمحتوى المحلي.

المرجع:

- https://developer.android.com/privacy-and-security/risks/webview-unsafe-file-inclusion

## 20.7 لا تعتبر HTTPS حماية لسلامة العميل

HTTPS يحمي البيانات أثناء النقل.

لكنه لا يحمي الصفحة من التطبيق الذي يملك بيئة العرض الخاصة بها.

## 20.8 أضف ضوابط fraud على جهة العميل والخادم

للخدمات المالية وعالية الحساسية:

```text
Application attestation
Device-risk signals
Runtime integrity checks
Transaction signing / binding
Behavioral fraud detection
Step-up authentication
Independent transaction confirmation
Session anomaly detection
```

---

# 21. منظور الكشف

APIs ذات الإشارة المهمة تشمل:

```text
setJavaScriptEnabled(true)
addJavascriptInterface(...)
evaluateJavascript(...)
loadUrl(dynamicValue)
@JavascriptInterface
setAllowUniversalAccessFromFileURLs(true)
```

كل واحدة منفردة قد تكون شرعية.

يكون الكشف أكثر دلالة عند تحليل العلاقة بين هذه APIs:

```text
Dynamic Web Content
      ↓
JavaScript Execution
      ↓
Native Bridge
      ↓
DOM / Runtime Injection
      ↓
Sensitive Native Capability
      ↓
Storage / Network / Permission Workflow
```

ينبغي للأدوات الأمنية إعادة بناء **مسار الثقة** بدل وضع تحذير على API معزولة.

---

# 22. التقييم البحثي

استنادًا إلى العينة واستخبارات التهديدات العامة:

1. **نمط Web-to-Native حقيقي أمنيًا وذو صلة عملية.**
2. **يمكن رصد موقع شرعي والتفاعل معه محليًا داخل WebView عدائي دون اختراق الخادم.**
3. **توجد analogues مباشرة في banking malware مثل Brokewell وSturnus.**
4. **الإعلانات والشبكات الاجتماعية قنوات توزيع مثبتة لAndroid financial malware.**
5. **Accessibility وoverlays وWebViews وkeylogging والتحكم عن بعد أصبحت تُدمج بدل استخدامها منفردة.**
6. **السلالة الديناميكية في Android أقدم من SpyNote وSpyMax: remote-command/botnet-like control موثق علنًا منذ 2010، post-install downloadable capability منذ 2011، targeted RAT architecture منذ 2012، وautomatic script updating منذ 2013؛ الأجيال اللاحقة مثل SpyNote/SpyMax تمثل consolidation وcommoditization لا بداية التحكم الديناميكي.**
7. **يمكن لنمط WebView-to-Native أن يدعم آثار احتيال عالية؛ ويستخدم البحث Trust Conversion كإطار تحليلي لا taxonomy رسمية ولا ranking عالمي.**
8. **المشكلة الدفاعية تتجاوز حماية الخوادم إلى حماية ثقة العميل والجلسات والمعاملات وبيئة التنفيذ على الجهاز.**
9. **نموذج تهديد تالٍ معقول هو AI-adaptive trust manipulation حيث تؤثر behavioral telemetry في توقيت ومحتوى التفاعل العدائي؛ وهذا Forecast وليس قدرة مثبتة في العينة.**

---

## 22.1 نموذج التهديد المستقبلي — من Static Trust Hijacking إلى AI-Adaptive Trust Manipulation

### الحالة: Research Assessment / Forecast

العينة المحللة **لا تثبت** وجود AI agent يتحكم في حلقة الهجوم.

لكن primitives الموجودة يمكن أن تدعم مستقبلًا بنية تكيفية إذا تم ربطها بطبقة قرار تعتمد على AI.

الفارق المهم هو بين **emotion recognition** و**behavioral inference**.

أحداث مثل:

```text
focus / focusin
input
keydown / keyup
blur
click
resize
navigation state
```

لا تكشف بصورة موثوقة الحالة الشعورية الداخلية للإنسان.

لكن يمكن أن تكون **behavioral proxies** لظواهر قابلة للملاحظة مثل:

```text
hesitation
repeated input correction
abandonment
navigation reversal
interaction speed
workflow stage
response to prompts
```

وبذلك يمكن تصور نموذج تكيفي مستقبلي:

```text
تجربة ويب شرعية / مقنعة
        ↓
Interaction telemetry
        ↓
Behavioral state estimation
        ↓
AI / agent decision layer
        ↓
اختيار استراتيجية التفاعل التالية
        ↓
تعديل المحتوى على جهة العميل أو توقيت Native workflow
        ↓
Outcome observation
        ↓
Policy adaptation
```

المغزى الأمني ليس أن AI يستطيع "قراءة المشاعر".

بل أن المهاجم قد لا يحتاج بعد الآن إلى كتابة كل قرار كقاعدة ثابتة.

يمكن لطبقة القرار أن تختار مثلًا:

```text
الإبقاء على الصفحة الطبيعية
تأخير فعل مريب
تغيير المحتوى التوضيحي
تعديل توقيت مسار Native
إيقاف التفاعل إذا ظهرت مقاومة
```

وهذا يحول social engineering ثابتة إلى **closed-loop adaptive interaction system**.

### لماذا هذا التوقع معقول تقنيًا؟

توثق التقارير العامة بالفعل تطورات مجاورة:

1. **AI-assisted social engineering أصبح تشغيليًا.**  
   تقارير Mandiant / Google Threat Intelligence Group توثق استخدام adversaries للـLLMs في social engineering أكثر تخصيصًا.

2. **Malware بدأ يستدعي LLM أثناء التنفيذ.**  
   تقارير M-Trends 2026 وأبحاث Google عن AI توثق عائلات مثل PROMPTFLUX وPROMPTSTEAL تستدعي نماذج أثناء التشغيل لتوليد كود أو أوامر ديناميكية.

3. **Agentic offensive behavior بدأ بالظهور.**  
   تصف أبحاث Google في 2026 انتقال 2025 من استخدام AI كمساعد إنتاجية إلى أدوات تكيفية ووكلاء يعملون بتدخل بشري أقل.

4. **Android financial malware الحديث يملك أصلًا primitives غير AI اللازمة.**  
   الحملات العامة توثق WebViews وoverlays وAccessibility وkeylogging وremote control وdynamic C2 وDevice Takeover.

الخطوة المتبقية هي دمج telemetry والتحكم مع real-time decision model.

### ما هو Observed وما هو Forecast؟

```text
OBSERVED / PUBLICLY DOCUMENTED

WebView instrumentation                     ✓
Native JavaScript bridges                   ✓
DOM / credential collection                 ✓
Accessibility abuse                         ✓
Overlay attacks                             ✓
Remote device control                       ✓
Dynamic C2 instructions                     ✓
AI-assisted social engineering              ✓
Malware querying LLMs during execution      ✓
Agentic / adaptive offensive tooling        ✓


NOT ESTABLISHED BY THIS RESEARCH

DOM telemetry
      ↓
AI behavioral model
      ↓
autonomous live WebView adaptation
      ↓
permission-workflow optimization
      ↓
closed-loop Android financial fraud
                                                ?
```

لذلك يجب التعامل مع الحلقة الأخيرة كـ**Future Threat Model**، وليس كحقيقة أن banking malware عامة تطبقها بالفعل على نطاق واسع.

### الدلالة الدفاعية

الحماية الثابتة على جهة العميل تصبح أضعف عندما يستطيع المهاجم تغيير سلوكه استجابة للبيئة أو المستخدم.

ينبغي افتراض أن العملاء العدائيين مستقبلًا قد يحسنون استراتيجية التفاعل ديناميكيًا، وبالتالي يجب إعطاء الأولوية لضوابط لا تعتمد فقط على شكل UI أو تسلسلها.

من المبادئ الدفاعية:

```text
independent transaction confirmation
server-side behavioral fraud analytics
device and application integrity signals
session anomaly detection
strong origin separation
minimal Web-to-Native capability
runtime detection of Accessibility / overlay abuse
high-risk action binding to trusted application state
```

والتقييم البحثي:

> **التطور المرجح التالي ليس فقط phishing content مولدًا بالذكاء الاصطناعي؛ بل AI-assisted orchestration لتفاعل العميل نفسه، بحيث تستخدم behavioral signals لتحديد متى وكيف يتم التلاعب ببيئة عميل عدائية.**

هذا التوقع متسق مع الانتقال الأوسع من malware ثابت إلى tooling تكيفي وagentic، لكن الحلقة الكاملة الخاصة بـAndroid ما تزال **غير مثبتة ضمن الأدلة التي تمت مراجعتها**.


---

# 23. المراجع

## Android / Platform

1. [Android Developers — WebView: Native Bridges](https://developer.android.com/privacy-and-security/risks/insecure-webview-native-bridges)
2. [Android Developers — Build Web Apps in WebView](https://developer.android.com/develop/ui/views/layout/webapps/webview)
3. [Android Developers — WebViews: Unsafe File Inclusion](https://developer.android.com/privacy-and-security/risks/webview-unsafe-file-inclusion)
4. [Android Developers — WebSettings API](https://developer.android.com/reference/android/webkit/WebSettings)
5. [Android Developers — Access Native APIs with JavaScript Bridge](https://developer.android.com/develop/ui/views/layout/webapps/native-api-access-jsbridge)
6. [Android Developers — WebView API Reference](https://developer.android.com/reference/android/webkit/WebView)
7. [AndroidX WebKit — WebViewCompat](https://developer.android.com/reference/androidx/webkit/WebViewCompat)
8. [Android Developers — Security Checklist](https://developer.android.com/privacy-and-security/security-tips)
9. [Google Security Blog — What's New in Android Security and Privacy in 2025](https://blog.google/security/whats-new-in-android-security-privacy-2025/)
10. [Google Security Blog — Keeping Google Play & Android App Ecosystems Safe in 2025](https://blog.google/security/keeping-google-play-android-app-ecosystem-safe-2025/)
## التاريخ المبكر للتحكم الديناميكي في Android

11. [SecurityWeek — Sophisticated New Android Trojan “Geinimi” Spreading in China](https://www.securityweek.com/sophisticated-new-android-trojan-geinimi-spreading-china/)
12. [University of New Brunswick — Android Botnet Dataset](https://www.unb.ca/cic/datasets/android-botnet.html)
13. [Ars Technica — DroidDream Light](https://arstechnica.com/gadgets/2011/06/droiddream-light-a-malware-nightmare-booted-from-android-market-encumbered-boobies-out-of-android-marketdroiddream-light-an-malware-nightware-booted-from-android-market/)
14. [Trend Micro — Adding Android and Mac OS X Malware to the APT Toolbox](https://documents.trendmicro.com/assets/wp/wp_adding-android-and-mac-osx-malware-to-the-apt-toolbox.pdf)
15. [SecurityWeek — Hackers Sell APK Binders for Google Android Remote Access Tool](https://www.securityweek.com/hackers-sell-apk-binders-google-android-remote-access-tool/)
16. [Trend Micro — Malicious and High-Risk Android Apps Hit 1 Million](https://www.trendmicro.com/vinfo/us/security/news/mobile-safety/malicious-and-high-risk-android-apps-hit-1-million-where-do-we-go-from-here)
17. [SecurityWeek — Developers of Android RAT DroidJack Traced to India](https://www.securityweek.com/developers-android-rat-droidjack-traced-india/)
## السلالة التاريخية لـBanking / RAT

18. [CERT Polska — GMBot: Android “poor man's webinjects”](https://cert.pl/posts/2015/10/lang_plgmbot-androidowa-ubozsza-wersja-webinjectowlang_pllang_engmbot-android-poor-mans-webinjectslang_en-2/)
19. [Mandiant / Google Cloud — Android Overlay Malware Spreading via SMS Phishing in Europe](https://cloud.google.com/blog/topics/threat-intelligence/latest-android-overlay-malware-spreading-in-europe/)
20. [Palo Alto Networks — SpyNote reported in 2016](https://www.paloaltonetworks.com/blog/2016/07/palo-alto-networks-news-of-the-week-july-30-2016/)
21. [MITRE ATT&CK — SpyNote RAT (S0305)](https://attack.mitre.org/software/S0305/)
22. [Group-IB — Craxs RAT, from SpyMax lineage to banking-fraud tooling](https://www.group-ib.com/blog/craxs-rat-malware/)
23. [UNODC — Developments in Cyber-Enabled Fraud and Technological Innovation (2024)](https://www.unodc.org/roseap/uploads/documents/Publications/2024/TOC_Convergence_Report_2024.pdf)
24. [ERNW / Insinuator — SpyMax Technical Analysis](https://insinuator.net/2022/09/spymax-the-android-rat-and-it-works-like-that/)
25. [ThreatFabric — S.O.V.A.](https://www.threatfabric.com/blogs/sova-new-trojan-with-fowl-intentions)
26. [ThreatFabric — Xenomorph](https://www.threatfabric.com/blogs/xenomorph-a-newly-hatched-banking-trojan)
27. [CYFIRMA — Unmasking EVLF DEV / CraxsRAT](https://www.cyfirma.com/research/unmasking-evlf-dev-the-creator-of-cypherrat-and-craxsrat/)
28. [Broadcom — CraxsRAT](https://www.broadcom.com/support/security-center/protection-bulletin/craxsrat)
## Android financial malware الحديثة / Threat Intelligence

29. [ThreatFabric — Brokewell](https://www.threatfabric.com/blogs/brokewell-do-not-go-broke-by-new-banking-malware)
30. [ThreatFabric — Sturnus](https://www.threatfabric.com/blogs/sturnus-banking-trojan-bypassing-whatsapp-telegram-and-signal)
31. [ThreatFabric — Exposing Crocodilus](https://www.threatfabric.com/blogs/exposing-crocodilus-new-device-takeover-malware-targeting-android-devices)
32. [ThreatFabric — Crocodilus: Evolving Fast, Going Global](https://www.threatfabric.com/blogs/crocodilus-mobile-malware-evolving-fast-going-global)
33. [ThreatFabric — New TrickMo Variant](https://www.threatfabric.com/blogs/new-trickmo-variant-device-take-over-malware-targeting-banking-fintech-wallet-auth-app)
34. [ThreatFabric — StreamRat: From Meta Ads to Full Device Takeover](https://www.threatfabric.com/blogs/from-meta-ads-to-full-device-takeover-uncovering-streamrat)
35. [Cleafy — PlayPraetor](https://www.cleafy.com/cleafy-labs/playpraetors-evolving-threat-how-chinese-speaking-actors-globally-scale-an-android-rat)
36. [Cleafy — ToxicPanda](https://www.cleafy.com/cleafy-labs/toxicpanda-a-new-banking-trojan-from-asia-hit-europe-and-latam)
37. [Cleafy — Medusa Reborn](https://www.cleafy.com/cleafy-labs/medusa-reborn-a-new-compact-variant-discovered)
38. [ESET — Phishing in PWA / WebAPK Applications](https://www.welivesecurity.com/en/eset-research/be-careful-what-you-pwish-for-phishing-in-pwa-applications/)
39. [ESET — Threat Report H2 2024](https://www.welivesecurity.com/en/eset-research/eset-threat-report-h2-2024/)
## AI / Adaptive Offensive Tooling

40. [Google Cloud / Mandiant — AI Risk and Resilience (2026)](https://cloud.google.com/security/resources/ai-risk-and-resilience)
41. [Google Cloud / Mandiant — M-Trends 2026 Executive Edition](https://cloud.google.com/security/resources/m-trends-executive-edition)
42. [Google Cloud / GTIG — AI Threat Tracker: Advances in Threat Actor Usage of AI Tools](https://cloud.google.com/blog/topics/threat-intelligence/threat-actor-usage-of-ai-tools)
## Mobile Security Testing / Defensive Standards

43. [OWASP MAS — MASTG-KNOW-0018: WebViews](https://mas.owasp.org/MASTG/knowledge/android/MASVS-PLATFORM/MASTG-KNOW-0018/)
44. [OWASP MAS — MASTG-TEST-0334: Native Code Exposed Through WebViews](https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0334/)
45. [OWASP MAS — MASTG-BEST-0035: Prefer Origin Scoped Messaging Over Legacy JavaScript Bridges](https://mas.owasp.org/MASTG/best-practices/MASTG-BEST-0035/)
46. [OWASP MAS — MASTG-TEST-0252: References to Local File Access in WebViews](https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0252/)
47. [OWASP MAS — MASTG-BEST-0058: Restrict Native Functionality Exposed Through WebView Bridges](https://mas.owasp.org/MASTG/best-practices/MASTG-BEST-0058/)
---

# 24. الحد النهائي للإفصاح

يتعمد هذا المنشور حذف operational payloads، ومنطق التحكم بالحملات، وtarget-specific element mappings، والتعليمات خطوة بخطوة للاستغلال.

راجعت النسخة النهائية سجلات سلامة العينة المحفوظة بشكل خاص، ومسارات الكود الثابت الرئيسية، ومسار المعالجة/الإرسال، وإرشادات Android الحالية، والادعاءات التاريخية واستخبارات التهديدات الرئيسية.

الادعاءات مقيدة عمدًا بنموذج الأدلة في هذا البحث. التشابه مع عائلة malware أخرى لا يعني مؤلفًا مشتركًا أو بنية مشتركة أو direct code lineage إلا إذا أثبت مصدر مستقل ذلك.

الهدف هو توثيق:

```text
فشل حد الثقة
الأثر الأمني المرصود
العلاقة مع malware مالي حقيقي
شروط الاستغلال
المعمارية الدفاعية المطلوبة
```

الدرس المركزي:

> **يمكن أن يبقى الموقع الشرعي آمنًا على مستوى الخادم بينما يتم اختطاف تفاعل العميل الموثوق داخل عميل Android عدائي.**

والمبدأ الدفاعي المقابل:

> **لا تسمح لثقة الويب بأن تصبح ثقة Native دون حد سياسة صريح، واعٍ بالـorigin، ومحدود القدرات.**

---

## حدود التعامل مع المادة البحثية والنشر

تم تصميم هذا المنشور لتوثيق النتائج الدفاعية دون إعادة توزيع العينة غير العامة أو نسبة مؤلف غير مثبت.

ويستبعد عمدًا:

```text
أرشيفات المصدر الخاصة
الكود الكامل غير المنشور
معرفات شخصية
هويات المشغلين
تفاصيل البنية الخاصة
تعليمات استغلال مرتبطة بهدف
ادعاءات attribution غير مثبتة
```

تُحتفظ سجلات سلامة المادة وتفاصيل provenance غير العامة بشكل منفصل للمراجعة المصرح بها والإفصاح المسؤول.

هذه وثيقة بحث تقني، وليست استشارة قانونية ولا تقرير attribution.

---

# الملحق A — مصفوفة الأدلة والادعاءات

هذا الملحق هو طبقة التدقيق في البحث، ويربط كل ادعاء مادي بفئة الدليل التي تدعمه.

## A.1 تصنيفات الأدلة

| التصنيف | المعنى |
|---|---|
| **CONFIRMED — SAMPLE** | ملاحظ مباشرة في المصدر/العينة المحللة. |
| **CONFIRMED — PLATFORM** | سلوك موثق رسميًا من Android / Google. |
| **CORRELATED — THREAT INTEL** | سلوك موثق في malware أو fraud research عام؛ يدعم التشابه أو السياق التاريخي وليس authorship مشتركًا. |
| **ASSESSMENT** | استنتاج بحثي مشتق من عدة فئات أدلة. |
| **NOT ESTABLISHED** | ادعاء لا تكفي الأدلة التي تمت مراجعتها لإثباته. |

## A.2 التعامل مع المادة البحثية

اشتُقت النتائج التقنية من **عينة أمنية غير عامة خاصة بـAndroid** تمت مراجعتها لأغراض دفاعية.

لا يُنشر:

```text
الأرشيف الأصلي
أسماء classes الخاصة
package names
developer identifiers
internal naming conventions
source hashes
provenance indicators
```

تنشر الورقة فقط الحد الأدنى الضروري لفهم النمط:

```text
Android APIs عامة
إعدادات ذات صلة أمنية
Web-to-Native data flows مجردة
permission-workflow patterns مجردة
منطق دفاعي للكشف
إرشادات التخفيف
```

تم حذف أو تعميم implementation-specific identifiers لأنها غير ضرورية للإصلاح وقد تكشف provenance أو attribution لا علاقة لهما بجوهر البحث.

لا تنسب هذه الورقة العينة إلى عائلة malware أو مطور أو operator أو research group أو historical lineage محدد.

وأي نقاش تاريخي لعائلات عامة يعتمد على تقارير مستقلة منشورة ويُستخدم للمقارنة وليس لإسناد العينة.

تُحتفظ تفاصيل المصدر والبصمات والمعرفات الدقيقة وchain-of-custody بشكل خاص للمراجعة المصرح بها والإفصاح المسؤول عند الحاجة.

## A.3 Claim Matrix

| ID | الادعاء العام | أساس الدليل العام | التصنيف |
|---|---|---|---|
| **E-01** | يمكن تهيئة WebView لتحميل عناوين شبكة ديناميكية مع بقائها داخل التطبيق. | ملاحظ في العينة غير العامة باستخدام APIs تحميل URL القياسية في Android WebView. | **CONFIRMED — SAMPLE** |
| **E-02** | يمكن أن يجتمع تنفيذ JavaScript وNative JavaScript Bridge في WebView نفسه. | ملاحظ باستخدام `setJavaScriptEnabled(...)` و`addJavascriptInterface(...)`. | **CONFIRMED — SAMPLE + PLATFORM** |
| **E-03** | إعدادات الوصول الواسع للملفات/المحتوى تزيد حساسية WebView المزود بجسر. | ملاحظ في العينة؛ Android توثق مخاطر file/content access المتساهل. | **CONFIRMED — SAMPLE + PLATFORM** |
| **E-04** | يمكن تنفيذ JavaScript وقت التشغيل داخل مستند WebView الحالي. | ملاحظ باستخدام `evaluateJavascript(...)`. | **CONFIRMED — SAMPLE** |
| **E-05** | يمكن استخدام lifecycle الصفحة لتشغيل منطق Instrumentation للصفحة بعد اكتمال التنقل. | ملاحظ عبر page-completion handling في WebView. | **CONFIRMED — SAMPLE** |
| **E-06** | يمكن لـJavaScript محقونة مراقبة أحداث إدخال وتفاعل DOM. | ملاحظ عبر listeners على input-oriented DOM events. | **CONFIRMED — SAMPLE** |
| **E-07** | يمكن لقيم مشتقة من DOM العبور من JavaScript إلى Native Android عبر Bridge. | ملاحظ في العينة؛ وAndroid توثق هذا السلوك لدوال bridge الموسومة. | **CONFIRMED — SAMPLE + PLATFORM** |
| **E-08** | يمكن لبيانات قادمة من الويب دخول مسار Native للمعالجة / الإرسال. | يوجد static processing/send path في المصدر؛ لا تدعي الورقة runtime network reachability من source review وحده. | **CONFIRMED — SAMPLE (STATIC PATH)** |
| **E-09** | يمكن لتفاعل الويب بدء مسارات Native للصلاحيات أو القدرات. | ملاحظ عبر Web-to-Native capability path؛ ويبقى authorization النهائي بيد Android/المستخدم. | **CONFIRMED — SAMPLE** |
| **E-10** | يمكن وضع قدرات Android حساسة خلف Web-to-Native workflows. | تشير العينة إلى camera/location/microphone/call logs/contacts/SMS/phone/files/overlay/all-files/Accessibility/battery settings. | **CONFIRMED — SAMPLE** |
| **E-11** | `addJavascriptInterface()` القديم لا يملك origin-based access control ويمكن أن يكون مكشوفًا للframes/iframes. | توثيق Android الرسمي. | **CONFIRMED — PLATFORM** |
| **E-12** | يمكن رصد موقع شرعي والتفاعل معه محليًا داخل WebView يملكه التطبيق دون إثبات server compromise. | يدعمه تركيب WebView + runtime script + DOM + bridge، وترتبط به أمثلة عامة مصرفية. | **CONFIRMED — SAMPLE CAPABILITY + CORRELATED — THREAT INTEL** |
| **E-13** | توجد أنماط WebView/native-bridge لجمع البيانات في banking malware حقيقية. | تقارير عامة عن Xenomorph وSturnus وغيرها. | **CORRELATED — THREAT INTEL** |
| **E-14** | malvertising/social advertising قنوات acquisition حقيقية لAndroid financial malware. | تقارير ESET وThreatFabric وغيرها. | **CORRELATED — THREAT INTEL** |
| **E-15** | يمكن للأدوات المكركة أن تتحول إلى hostile supply chain. | تقارير عن backdoored cracked tooling وتسريب builders. | **CORRELATED — THREAT INTEL** |
| **E-16** | التحكم الديناميكي في Android أقدم من SpyNote وSpyMax. | Geinimi 2010، DroidDream 2011، Luckycat 2012، وKSAPP 2013. | **CORRELATED — THREAT INTEL / HISTORICAL ASSESSMENT** |
| **E-17** | SpyNote/SpyMax نقاط consolidation مهمة وليستا أصل dynamic Android malware. | SpyNote ظاهر علنًا في 2016، SpyMax موثق نحو 2019، بينما التحكم الديناميكي موثق قبل ذلك بسنوات. | **CORRELATED — THREAT INTEL / ASSESSMENT** |
| **E-18** | 2024–2026 مرحلة تقارب وتوسع حديثة وليست ميلاد التقنيات الأساسية. | تقييم مقارن عبر حملات موثقة. | **ASSESSMENT** |
| **E-19** | يمكن لنمط WebView-to-Native دعم آثار احتيال عالية؛ وTrust Conversion framing تحليلي لا taxonomy رسمية. | تقييم يجمع العينة والمنصة واستخبارات التهديدات. | **ASSESSMENT** |
| **E-20** | البحث لا يثبت website-server compromise أو universal Android RCE أو automatic permission granting أو self-propagation. | لا يظهر هذا المسار في الأدلة المراجعة. | **NOT ESTABLISHED** |
| **E-21** | AI-adaptive WebView manipulation اعتمادًا على behavioral telemetry نموذج تهديد مستقبلي معقول، لكن الحلقة الكاملة غير مثبتة في العينة. | Forecast يدعمه وجود telemetry/control primitives في العينة مع تقارير عامة عن runtime LLM malware وadaptive offensive AI. | **ASSESSMENT / NOT ESTABLISHED AS OBSERVED SAMPLE BEHAVIOR** |

## A.4 دليل المنصة حول iframe/origin

توضح إرشادات Android الحالية أن `addJavascriptInterface()` القديم:

- متاح افتراضيًا لكل frame في WebView بما في ذلك `iframe`؛
- لا يملك origin-based access control؛
- لا يوفر طريقة آمنة لمعرفة URL للـframe الذي استدعى الواجهة.

المرجع الأساسي:

- Android Developers — **Access native APIs with JavaScript bridge**  
  https://developer.android.com/develop/ui/views/layout/webapps/native-api-access-jsbridge

مراجع إضافية:

- Android Developers — **WebView API reference**  
  https://developer.android.com/reference/android/webkit/WebView
- Android Developers — **Build web apps in WebView**  
  https://developer.android.com/develop/ui/views/layout/webapps/webview
- Android Developers — **Security checklist**  
  https://developer.android.com/privacy-and-security/security-tips
- OWASP MAS — **MASTG-TEST-0334: Native Code Exposed Through WebViews**  
  https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0334/
- OWASP MAS — **MASTG-BEST-0035: Prefer Origin Scoped Messaging Over Legacy JavaScript Bridges**  
  https://mas.owasp.org/MASTG/best-practices/MASTG-BEST-0035/

---

# الملحق B — حدود الأدلة

يحدد هذا الملحق ما يجوز وما لا يجوز استنتاجه من الأدلة المراجعة.

## B.1 مثبت / مدعوم مباشرة

```text
✓ يستطيع WebView المحلل تحميل network URLs.
✓ JavaScript مفعلة.
✓ يوجد Native object مكشوف عبر addJavascriptInterface().
✓ يوجد runtime JavaScript execution/injection.
✓ page lifecycle يشغل instrumentation logic.
✓ تتم مراقبة DOM input وinteraction events.
✓ يمكن لقيم الحقول العبور إلى Native Java.
✓ يوجد processing / data-send path في المصدر.
✓ يمكن لتفاعل الويب بدء permission-related native workflows.
✓ يشير منطق الصلاحيات إلى عدة قدرات Android حساسة.
✓ legacy addJavascriptInterface() مكشوف للframes بما فيها iframes.
✓ legacy addJavascriptInterface() لا يملك origin-based access control.
```

## B.2 مدعوم بارتباطات واقعية مستقلة

لا تُنسب النقاط التالية إلى العينة فقط لأنها تظهر في malware أخرى؛ بل تستخدم كإثبات أن التقنيات تعمل تشغيليًا في العالم الحقيقي:

```text
✓ Legitimate-site WebView/session abuse موجود في banking malware.
✓ WebView + JavaScript + native bridge data collection موجود.
✓ Accessibility + overlays + keylogging + remote control يتم دمجها.
✓ Social-media advertising / malvertising تستخدم لاكتساب الضحايا.
✓ Cracked RAT ecosystems قد تنشر tooling مزروعة وتسرع code reuse.
```

ومن العائلات/الحملات المذكورة:

```text
S.O.V.A.
Xenomorph
Brokewell
Crocodilus
PlayPraetor
Sturnus
TrickMo
StreamRat
SpyNote / SpyMax / CraxsRAT lineage
```

التشابه **لا يثبت**:

```text
shared authorship
shared infrastructure
shared implementation code
direct lineage
the same operator
the same campaign
```

إلا إذا أثبت مصدر خارجي ذلك بشكل مستقل.

## B.3 غير مثبت بالأدلة المراجعة

لا تثبت الأدلة المراجعة أيًا من الآتي:

```text
✗ اختراق خادم الموقع الشرعي
✗ Server-side RCE
✗ تعديل الموقع الشرعي لزوار آخرين
✗ Universal Android sandbox escape
✗ Automatic Android permission granting
✗ الإصابة عبر كل إعلان
✗ Self-propagation بين المواقع
✗ اختراق مستخدمين لم يشغلوا التطبيق العدائي
✗ تاريخ أصل وحيد للتقنية
✗ مؤلف أو مطور واحد يفسر كامل السلالة التطورية بين 2010 و2026
✗ حلقة AI كاملة تحول DOM behavior إلى permission optimization تلقائي
✗ Remote provenance/control لمصدر runtime script من source snapshot وحده
```

## B.4 ادعاءات تحتاج تحققًا إضافيًا

| السؤال | الدليل المطلوب |
|---|---|
| هل يمكن رصد موقع هدف محدد والتفاعل معه بصورة قابلة لإعادة الإنتاج؟ | Controlled reproduction على origin مصرح به مع WebView/network traces. |
| هل يمكن لiframe إعلاني معين الوصول للجسر؟ | اختبار frame-level مصرح به داخل نفس WebView المزود بالجسر. |
| هل يمكن إكمال permission workflow محددة عبر الرحلة المحفزة من الويب؟ | إعادة إنتاج خاصة بإصدار الجهاز توثق كل تفاعل للمستخدم/النظام. |
| هل يستقبل backend محدد البيانات؟ | Packet capture مصرح أو server logs أو protocol analysis. |
| هل عائلتا malware مرتبطتان مباشرة؟ | Code similarity أو infrastructure overlap أو artifacts أو source lineage أو attribution موثوق. |

## B.5 حدود التفسير

تدعم الأدلة ثلاثة استنتاجات منفصلة:

1. **قدرة العينة:** تحتوي العينة المحللة على مسار Web-to-Native قادر على رصد الصفحة محليًا، ونقل بيانات مشتقة من الويب إلى Native، وبدء مسارات Native حساسة.
2. **الارتباط المستقل:** توثق استخبارات التهديدات العامة أنماطًا تشغيلية قريبة في Android financial malware.
3. **فشل الثقة على جهة العميل:** يمكن أن يبقى الموقع الشرعي سليمًا على مستوى الخادم بينما يتم التلاعب بتفاعل المستخدم داخل hostile client environment.

ولا تثبت هذه الاستنتاجات اختراق الخادم، أو قدرة عامة لكل إعلان على التحكم بالجهاز، أو منح Accessibility تلقائيًا، أو ابتكار النمط بواسطة عائلة malware واحدة.

لذلك يفصل التفسير بين أربع طبقات:

- **Capability** — ما تسمح به البنية أو ما يظهر في المصدر الذي تمت مراجعته.
- **Reachability** — ما يمكن الوصول إليه تحت شروط تشغيل محددة.
- **Real-world correlation** — ما تثبته حملات مستقلة في الواقع العملي.
- **Attribution** — نسبة التطوير أو التشغيل أو الأصل إلى جهة محددة.

---

# الملحق C — قائمة إعادة الإنتاج والإفصاح

يتطلب التحقق الخاص بتطبيق معين، والمبني على هذا البحث، الأدلة التالية قبل تحديد exploitability أو severity:

```text
[ ] إصدار التطبيق / build بدقة
[ ] Android OS وWebView version
[ ] URL/origin المحدد
[ ] هل navigation يسيطر عليها المستخدم أم remote أم ثابتة؟
[ ] هل bridge موجودة على الصفحة/frame ذات الصلة؟
[ ] الدالة @JavascriptInterface المكشوفة
[ ] الأثر الأمني Native الذي تم الوصول إليه
[ ] تفاعل المستخدم المطلوب
[ ] Permission / Settings confirmation المطلوبة
[ ] Network destination إذا تم الادعاء بانتقال بيانات
[ ] فيديو / logs / packet trace حيث يكون الاختبار مصرحًا
[ ] Expected behavior vs observed behavior
[ ] Mitigation validation بعد الإصلاح
```

ويفصل تقييم الشدة بين:

```text
Architecture exists
        ≠
Exploit path is reachable
        ≠
Sensitive impact is reproducible
        ≠
Critical severity is justified
```

يجب أن يبقى هذا الفرق صريحًا في Responsible Disclosure.
