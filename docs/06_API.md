# 06. نقاط الاتصال الخارجية (External Integrations)

## Supabase (REST/PostgREST)
- **الاتصال:** نداءات `fetch` مباشرة (بلا SDK) لـ `https://qhhqodxbhhvhkjtzmmsh.supabase.co/rest/v1/...`.
- **المصادقة:** `apikey` + `Authorization: Bearer <anon key>` ثابتَين في الكود — هذا **متعمَّد وآمن** (نفس نمط `mamelon-erp`)؛ مفتاح `anon` مصمَّم أصلاً ليكون علنياً، والحماية الفعلية تأتي من سياسات RLS على مستوى القاعدة، لا من إخفاء المفتاح.
- **الجداول:** راجع `03_DATABASE.md`.

## بوت تليجرام
- **الآلية (منذ 25 أغسطس 2026، راجع ADR-C03):** العميل (`index.html`) **لا يتصل بتليجرام مباشرة إطلاقاً**. بدلاً من ذلك يستدعي `POST ${SUPABASE_URL}/functions/v1/telegram-notify` (دالة Supabase Edge Function) بنفس ترويسات المصادقة المستخدَمة لباقي نداءات Supabase (`apikey`, `Authorization: Bearer <anon key>`). الدالة نفسها هي اللي تتصل بـ `api.telegram.org` من جهة الخادم، وتقرأ التوكن من متغيّر بيئة سرّي (`TELEGRAM_BOT_TOKEN`) لا يظهر أبداً في أي كود يصل لمتصفح المستخدم.
- **الاتجاه:** إرسال فقط (Send-only) — لا استقبال، لا أوامر، لا Webhook وارد.
- **الوجهة:** مجموعة تليجرام واحدة ثابتة (`TELEGRAM_CHAT_ID`، سرّ خادم أيضاً — `chat_id` واحد لكل الإشعارات، بلا تمييز بين طبيب/عيادة).

### ✅ ثغرة توكن البوت المكشوف — مُغلَقة (25 أغسطس 2026، ADR-C03)
كانت الثوابت `TELEGRAM_BOT_TOKEN`/`TELEGRAM_CHAT_ID` مكتوبة صراحة بكود العميل — أُزيلت نهائياً. راجع `ADR-C03` في `10_DECISIONS.md` للتفاصيل الكاملة.

**⚠️ خطوة تشغيلية متبقية على المستخدم (ليست ثغرة، مجرد إعداد ناقص):** يجب ضبط `TELEGRAM_BOT_TOKEN` و`TELEGRAM_CHAT_ID` كأسرار (Secrets) على دالة `telegram-notify` عبر لوحة تحكم Supabase (Project Settings → Edge Functions → Secrets) أو الـ CLI (`supabase secrets set`). **حتى تُضبَط، كل إشعار سيفشل بصمت** (الدالة تُرجِع `{"error":"Server not configured"}` بدل الإرسال الفعلي — لا يكسر تدفق التسليم/الإرجاع نفسه، فقط الإشعار لا يصل).

## Supabase Edge Function: `telegram-notify`
- **الغرض:** وسيط خادم بين `index.html` وتليجرام — يحجب توكن البوت عن كود العميل. راجع `ADR-C03`.
- **المشروع:** `qhhqodxbhhvhkjtzmmsh` (نفس مشروع Supabase).
- **الطلب:** `POST` بجسم `{"text": "..."}`. يتطلب `Authorization: Bearer <JWT صالح>` (`verify_jwt: true`) — مفتاح `anon` نفسه صالح لهذا الغرض.
- **الأسرار المطلوبة على الدالة (تُضبَط من طرف المستخدم فقط، عبر لوحة تحكم Supabase أو CLI — لا توجد أداة MCP لضبط الأسرار برمجياً، وهذا مقصود):**
  - `TELEGRAM_BOT_TOKEN`
  - `TELEGRAM_CHAT_ID`
- **الاستجابة:** تُعيد استجابة تليجرام الخام (نجاح أو فشل) بنفس كود الحالة HTTP.
- **الكود المصدري للدالة:** لا يوجد في هذا المستودع محلياً — منشور مباشرة على Supabase. راجع `ADR-C03` لنسخة كاملة من الكود وقت النشر.

## html5-qrcode
مكتبة مسح باركود/QR، محمَّلة عبر CDN (`unpkg.com`) — لا إعداد إضافي، تُستخدَم مباشرة عبر `startScanner()`.
