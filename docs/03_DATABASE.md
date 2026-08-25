# 03. جداول قاعدة البيانات المستخدَمة (Database Tables Used)

> **⚠️ هذا التطبيق لا يملك قاعدة بيانات خاصة به** — يتصل مباشرة بنفس مشروع Supabase الذي يستخدمه `mamelon-erp` (`qhhqodxbhhvhkjtzmmsh`). المخطط الكامل موثَّق في `mamelon-erp/docs/03_DATABASE.md` — هذا الملف يوثّق فقط **كيف يستخدم تطبيق المندوب** هذه الجداول تحديداً.

## `cases`
| العملية | الحقول المتأثرة |
| :--- | :--- |
| قراءة (بحث/سلة) | `id, case_number, clinic_name, doctor_name, patient_name, product_type` |
| تحديث عند الالتقاط (`confirmPickup`) | `status='out_for_delivery', current_holder=<اسم المندوب>, picked_up_at=<الآن>` |
| تحديث عند التسليم | `status='delivered', current_holder=null, picked_up_at=null` |
| تحديث عند الإرجاع | `status='ready', current_holder=null, picked_up_at=null` |

**فلتر الحماية من التعارض:** أي `PATCH` عند الالتقاط يُقيَّد بـ `current_holder=is.null` — يمنع مندوبَين من أخذ نفس الحالة في نفس اللحظة.

## `deliveries`
إدراج فقط (لا قراءة). عند تسليم فردي: صف واحد. عند تسليم دفعة (منذ 25 أغسطس 2026): صف واحد **لكل حالة في الدفعة**، بنفس `recipient_name` المشترك لكن كل صف بـ`delivery_stage` الخاص به.

## `returns`
إدراج فقط. `case_number`, `delegate_name`, `reason` (اختياري).

## `delegate_directory`
**قراءة فقط** — `view` مخصَّص للمصادقة، يُطابَق رمز PIN المُدخَل معه مباشرة عند تسجيل الدخول.

## ⚠️ سياسات RLS
هذا التطبيق يعتمد كلياً على سياسات RLS المضبوطة من طرف `mamelon-erp` (لا سياسات خاصة به). أي تغيير في RLS لجداول `cases`/`deliveries`/`returns` من `mamelon-erp` يؤثر مباشرة على قدرة هذا التطبيق على القراءة/الكتابة — راجع `mamelon-erp/docs/03_DATABASE.md` قبل تشخيص أي عطل "الحفظ يفشل بصمت".
