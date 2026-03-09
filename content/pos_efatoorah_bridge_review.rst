مراجعة وتعديل الموديول بالكامل: ``pos_efatoorah_bridge``
========================================================

تنبيه مهم
---------

لا يوجد كود الموديول داخل هذا المستودع، كما أن الوصول المباشر إلى GitHub من بيئة التنفيذ
مقيّد حاليًا (``CONNECT tunnel failed, response 403``). لذلك لا يمكن تنفيذ patch مباشر على
ملفات الموديول من هنا.

مع ذلك، هذه خطة **تعديل كاملة على مستوى الموديول** مع نقاط إصلاح دقيقة قابلة للتطبيق فور
توفير الكود.

الهدف من التعديل
----------------

- منع حالة "نجاح الربط + رد فارغ".
- إلزامية حفظ بيانات ZATCA المرجعية (UUID/QR/Status/ClearedAt).
- توحيد عقد الاستجابة بين middleware و Odoo.
- تحسين التتبع (observability) وسهولة التشخيص.

التعديلات المطلوبة على مستوى الموديول (Module-Wide)
----------------------------------------------------

1) طبقة الإعدادات ``settings / ir.config_parameter``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- إضافة/تدقيق مفاتيح:

  - ``zatca_api_base_url``
  - ``zatca_api_token``
  - ``zatca_timeout_seconds`` (افتراضي 30)
  - ``zatca_retry_count`` (افتراضي 2)
  - ``zatca_strict_success`` (افتراضي True)

- التحقق عند الحفظ:

  - منع حفظ إعدادات ناقصة.
  - اختبار endpoint صحة (health check) وإظهار نتيجة واضحة.

2) طبقة النماذج ``models``
~~~~~~~~~~~~~~~~~~~~~~~~~~

- إضافة حقول تتبع في ``account.move`` و/أو ``pos.order``:

  - ``zatca_submission_state``: draft/sent/accepted/rejected/error
  - ``zatca_uuid``
  - ``zatca_qr``
  - ``zatca_cleared_at``
  - ``zatca_error_code``
  - ``zatca_error_message``
  - ``zatca_request_payload`` (JSON نصي sanitized)
  - ``zatca_response_payload`` (JSON نصي sanitized)

- فرض قاعدة: لا تعتبر الفاتورة "مرفوعة" إلا عند وجود ``zatca_uuid`` وحالة قبول صريحة.

3) طبقة الخدمة ``services`` (الاتصال بالوسيط)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- إنشاء عميل API موحّد (مثال: ``ZatcaBridgeClient``) يقوم بـ:

  - timeout/retry/backoff.
  - تسجيل request_id لكل عملية.
  - إرجاع كائن نتيجة موحّد دائمًا.

- شكل نتيجة داخلي موصى به:

.. code-block:: python

   {
       "ok": bool,
       "http_status": int,
       "provider_status": "accepted|rejected|warning|error",
       "uuid": "...",
       "qr": "...",
       "cleared_at": "...",
       "errors": [{"code": "...", "message": "..."}],
       "raw": {...}
   }

- لا تسمح بإرجاع ``ok=True`` إذا ``uuid`` مفقود عند ``strict_success=True``.

4) طبقة بناء الحمولة ``payload builder``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- التحقق قبل الإرسال (pre-flight validation):

  - VAT Number موجود وصحيح.
  - مجاميع السطور = مجاميع الرأس.
  - ضريبة كل سطر متوافقة مع الإجمالي.
  - التاريخ/العملة/نوع المستند حسب متطلبات ZATCA.

- عند الفشل: لا ترسل الطلب أصلًا، وأعد خطأ وظيفي واضح للمستخدم.

5) طبقة parsing وحفظ الرد ``response handling``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- منع ``swallow exceptions`` بشكل كامل.
- أي خطأ parsing يجب أن:

  - يسجل ``zatca_submission_state='error'``.
  - يحفظ ``raw response``.
  - يعرض رسالة خطأ مفهومة في واجهة Odoo.

- mapping إلزامي:

  - ``uuid -> zatca_uuid``
  - ``qr -> zatca_qr``
  - ``status -> zatca_submission_state``
  - ``errors[] -> zatca_error_*``

6) الواجهات ``views`` وتجربة المستخدم
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- إضافة Smart Button: "ZATCA Logs".
- Banner حالة واضح:

  - Accepted (أخضر)
  - Rejected/Warning (برتقالي)
  - Error (أحمر)

- عرض سبب الرفض من ``errors[]`` بدل عبارة عامة "بدون بيانات".

7) ``security`` وخصوصية البيانات
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- إخفاء التوكنات والأسرار في logs.
- صلاحيات منفصلة لمن يقدر يشاهد ``raw payload``.
- ACL تمنع المستخدم العادي من الاطلاع على مفاتيح API.

8) ``cron/queue`` وإعادة المحاولة
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- Queue job لكل فاتورة بدل معالجة متزامنة داخل الطلب UI.
- إعادة محاولات ذكية فقط على أخطاء الشبكة/5xx.
- عدم إعادة الإرسال في حالات ``rejected`` إلا بعد تصحيح البيانات.

اختبارات إلزامية قبل اعتماد التعديل
------------------------------------

1. **Unit tests**

- اختبار validator للحمولة.
- اختبار parser مع 4 سيناريوهات: accepted / rejected / malformed / empty body.

2. **Integration tests**

- محاكاة middleware يعيد:

  - 200 مع ``uuid`` (نجاح)
  - 200 بدون ``uuid`` (يجب فشل منطقي)
  - 400 مع errors (رفض صحيح)
  - 500 (retry ثم فشل)

3. **UAT**

- فاتورة مبيعات + فاتورة مشتريات + مرتجع.
- التأكد من ظهور الحالة والـ UUID/QR في الواجهة وفي قاعدة البيانات.

ترتيب التنفيذ المقترح (Sprint واحد)
-----------------------------------

- اليوم 1: توحيد client + schema + logging.
- اليوم 2: validator + parser + strict success.
- اليوم 3: views + security + queue/retry.
- اليوم 4: tests + bugfix.
- اليوم 5: UAT + release.

أين تكمن المشكلة في حالتك الحالية؟
----------------------------------

بناءً على الوصف (الاتصال ناجح لكن الرد النهائي بدون بيانات)، فالأرجح:

- mismatch بين schema استجابة middleware وما يتوقعه parser في Odoo.
- أو ``HTTP 200`` يُعامل كنجاح رغم غياب الحقول المرجعية.
- أو payload غير مطابق وتم إخفاء سبب الرفض بدل إظهاره.

أول فحص حاسم: قارن ``raw response`` من middleware مع ما يُحفظ في حقول الفاتورة؛
إذا ظهر ``uuid`` في الرد الخام ولم يُحفظ، فالخلل في parsing/mapping داخل الموديول.
