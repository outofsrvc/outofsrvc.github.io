---
title: "Flare-On 2015: حل التحدي الثاني"
description: "نحل التحدي الثاني من تحديات Flare-On (2015): ملف PE بدون لاحقة نفحصه بأداة HxD، ثم تحليل ساكن عبر IDA Pro لفك خوارزمية تحقق معقدة (XOR + تدوير + جمع تراكمي) وكتابة سكريبت IDC لاستخراج كلمة السر."
date: 2026-08-13T09:00:00+03:00
slug: "flare-on15-2"
translationKey: "flare-on15-2"
categories: [reverse-engineering, ctf]
tags: [ctf, flare-on, x86, assembly, ida-pro, pe-format, hxd, idc, crypto]
ShowToc: true
draft: false
---

بسم الله الرحمن الرحيم.

في هذا المقال، سنحل التحدي الثاني من تحديات **Flare-On (2015)**. وكما هو معتاد، سنعتمد على منهجية التعلم بالممارسة (Learning-by-Doing) ونهج التعلم في الوقت المناسب (Just-in-Time Learning).

## 1. تحميل التحدي وفك الضغط

نقوم بتحميل التحدي من الموقع الرسمي: [flare-on.com](https://flare-on.com/)

> كلمة سر الملفات: **flare**.

بعد فك الضغط، نرى ملفاً **بدون لاحقة** (Extension)، لذلك نقوم بتحليله في أداة **HxD** لمعرفة لاحقته الأصلية:

![فحص الملف في HxD](/assets/img/flareon15-2/hxd.png)
_شكل (1): فحص الملف في HxD_

---

## 2. تحديد نوع الملف وتغيير لاحقته

كما نرى في الصورة، الملف من نوع **PE (Portable Executable)**، لذلك نقوم بتغيير لاحقته إلى `.exe`.

---

## 3. أول تشغيل وملاحظة السلوك

بعد تغيير اللاحقة، نقوم بتشغيل الملف عبر نافذة الـ CMD:

![تشغيل البرنامج](/assets/img/flareon15-2/cmd1.png)
_شكل (2): تشغيل البرنامج_

يظهر البرنامج رسالة:

```text
You crushed that last one! Let's up the game.
Enter the password>
```

**أي:** «لقد تخطيت التحدي السابق! لنرفع مستوى اللعبة»، ثم يطلب كلمة السر، وبعد إدخال أي شيء يعطينا النتيجة.

حسناً، لنتعمق في البرنامج ونقوم بتحليله استاتيكياً في **IDA Pro**.

---

## 4. التحليل الساكن: مقدمة غريبة

في بداية الكود نلاحظ **غلطاً مقصوداً** في الـ Prologue الخاص بالدالة `sub_401000`:

```asm
sub_401000 proc near
.text:00401000 pop     eax
.text:00401001 push    ebp
.text:00401002 mov     ebp, esp
```

الطبيعي أن تبدأ الدالة بـ `push ebp` لإنشاء إطار ستاك (Stack Frame) خاص بها، لكن هنا أُضيفت تعليمة `pop eax` **قبلها**. هذه التعليمة تسحب **عنوان العودة** (Return Address) من قمة الستاك — أو أي معامل كان قد دُفع قبل استدعاء الدالة — وتخزنه في `eax`، وسنعرف أهميته لاحقاً.

بالنظر إلى الكود، نرى أنه تم تعريف **مقبضَين (Handles)**: واحد خاص بالكتابة (Write) وواحد خاص بالقراءة (Read):

- **مقبض الكتابة** `[ebp-8]`: (stdout، الرسالة، `0x43`).
- **مقبض القراءة** `[ebp-0Ch]`: (stdin، `unk_402159`، `0x32`).

```asm
.text:00401004                 sub     esp, 10h
.text:00401007                 mov     [ebp-10h], eax
.text:0040100A                 push    0FFFFFFF6h        ; STD_INPUT_HANDLE
.text:0040100C                 call    GetStdHandle
.text:00401012                 mov     [ebp-0Ch], eax
.text:00401015                 push    0FFFFFFF5h        ; STD_OUTPUT_HANDLE
.text:00401017                 call    GetStdHandle
.text:0040101D                 mov     [ebp-8], eax
.text:00401020                 push    0
.text:00401022                 lea     eax, [ebp-4]
.text:00401025                 push    eax
.text:00401026                 push    43h               ; 'C'
.text:00401028                 push    offset aYouCrushedThat ; "You crushed that last one! Let's up the game."
.text:0040102D                 push    dword ptr [ebp-8]
.text:00401030                 call    WriteFile
.text:00401036                 push    0
.text:00401038                 lea     eax, [ebp-4]
.text:0040103B                 push    eax
.text:0040103C                 push    32h               ; '2'
.text:0040103E                 push    offset unk_402159
.text:00401043                 push    dword ptr [ebp-0Ch]
.text:00401046                 call    ReadFile
```

---

## 5. استدعاء دالة التحقق

بالنزول قليلاً، نرى **ست معاملات** دُفعت على الستاك، لكن تركيزنا يجب أن يكون على **ثلاث معاملات** فقط:

```asm
.text:0040104C                 push    0               ; lpOverlapped
.text:0040104E                 lea     eax, [ebp-4]
.text:00401051                 push    eax             ; lpNumberOfBytesWritten
.text:00401052                 push    11h             ; nNumberOfBytesToWrite
.text:00401054                 push    dword ptr [ebp-4]      ; عدد البايتات المقروءة
.text:00401057                 push    offset unk_402159      ; المُدخل من المستخدم
.text:0040105C                 push    dword ptr [ebp-10h]    ; المعامل السري (قيمة pop eax)
.text:0040105F                 call    sub_401084
.text:00401064                 add     esp, 0Ch
```

المعاملات الثلاثة المهمة:

1. `[ebp-10h]`: القيمة التي سُحبت أولاً بتعليمة `pop eax` — قد تكون مفتاح XOR، أو القيمة الصحيحة، أو عنوان العودة.
2. `unk_402159`: مدخلات المستخدم.
3. `[ebp-4]`: طول المدخلات الفعلي الذي أدخله المستخدم.

المعاملات الثلاثة الأخرى تبقى داخل الستاك ولا تؤثر على التنفيذ — **هي مجرد تشتيت فقط لا غير**.

بعدها ستقوم الدالة `sub_401084` بالتحقق من صحة العلم (Flag)، لأن ما بعد الاستدعاء يوجد اختبار (test) وقفز شرطي، والشرط هو `eax != 0`.

---

## 6. تحليل دالة التحقق sub_401084

لنتعمق في تحليل الدالة:

```asm
sub_401084 proc near
    push    ebp
    mov     ebp, esp
    sub     esp, 0          ; لا يخصص مساحة إضافية (مجرد إطار)
    push    edi
    push    esi
    xor     ebx, ebx        ; ebx = 0 (المجموع التراكمي bx)
    mov     ecx, 25h        ; ecx = 37 (عدد البايتات المطلوبة)
    cmp     [ebp+arg_8], ecx ; إذا كان طول المدخل أقل من 37 -> فشل
    jl      loc_4010D7
    mov     esi, [ebp+arg_4]  ; esi = مؤشر المدخل (input buffer)
    mov     edi, [ebp+arg_0]  ; edi = مؤشر البيانات المخفية (hidden buffer)
    lea     edi, [edi+ecx-1]  ; edi = edi + 36 (يشير إلى آخر عنصر في المخفي)
```

من الواضح أن الدالة تأخذ **3 معاملات** بنمط استدعاء **cdecl**. وبالعودة إلى الكود الذي يستدعيها، نرى ما هي المعاملات الثلاثة:

1. `arg_0 (ebp+8)` = `[ebp-10h]` → البيانات المخفية.
2. `arg_4 (ebp+0Ch)` = `unk_402159` → مدخلات المستخدم.
3. `arg_8 (ebp+10h)` = `[ebp-4]` → طول المدخلات.

أهم النقاط التي يجب التركيز عليها:

- `ebx = 0`: المجموع التراكمي.
- `ecx = 37`: عداد، غالباً لطول الحروف الصحيحة للـ Flag.
- `esi`: يشير إلى بداية مدخلات المستخدم.
- `edi`: يشير إلى نهاية المصفوفة المخفية `hidden[36]`.

---

## 7. الحلقة الرئيسية loc_4010A2

```asm
loc_4010A2:
    mov     dx, bx        ; dx = المجموع التراكمي السفلي (bx)
    and     dx, 3         ; dx = bx & 3 (نأخذ آخر بتّين فقط)
    mov     ax, 1C7h      ; ax = 0x1C7
    push    eax           ; حفظ eax على الستاك
    sahf                  ; تحميل ah (0x01) إلى أعلام EFLAGS -> تعيين CF = 1
    lodsb                 ; al = بايت من مدخل المستخدم (من esi)، ثم esi++
    pushf                 ; حفظ الأعلام الحالية على الستاك
    xor     al, [esp+4]   ; al = al XOR 0xC7
    xchg    cl, dl        ; تبادل cl مع dl (cl = العداد، dl = bx&3)
    rol     ah, cl        ; تدوير ah (0x01) بمقدار (bx&3) -> ah = 1 << (bx&3)
    popf                  ; استعادة الأعلام (CF = 1)
    adc     al, ah        ; al = al + ah + CF
    xchg    cl, dl        ; إعادة cl و dl إلى أماكنهما
    xor     edx, edx      ; edx = 0
    and     eax, 0FFh     ; eax = al فقط
    add     bx, ax        ; bx = bx + al (المجموع التراكمي)
    scasb                 ; قارن al مع [edi] ثم edi++ (DF=0)
    cmovnz  cx, dx        ; إذا لم تتساوَ (ZF=0): cx = dx = 0 -> فشل مبكر
    pop     eax           ; استعادة 0x1C7 من الستاك
    jecxz   loc_4010D7    ; إذا أصبح cx = 0 -> اخرج بفشل
    sub     edi, 2        ; edi = edi - 2 (مع زيادة scasb يصبح صافي edi--)
    loop    loc_4010A2    ; إنقاص ecx ثم القفز إذا لم يصل إلى 0
    jmp     short loc_4010D9 ; نجاح
```

لنفكك هذه التعليمات بعناية:

### 7.1 سر `xor al, [esp+4]`

```asm
push    eax          ; يخزن 0x1C7 على الستاك
sahf                ; ينقل ah (0x01) إلى البايت المنخفض من EFLAGS
lodsb               ; يقرأ بايت من المدخل إلى al
pushf               ; يدفع الأعلام الحالية إلى الستاك
```

الآن الستاك يحتوي على: `[esp]` = الأعلام، `[esp+4]` = القيمة `0x1C7` (بتمثيل little-endian: `C7 01 00 00`). إذاً `[esp+4]` = `0xC7`:

```asm
xor al, 0xC7
```

### 7.2 قوة التدوير R

```asm
xchg    cl, dl      ; cl = العداد، dl = bx&3  -> بعد التبادل: cl = bx&3
rol     ah, cl      ; ah = 1 << (bx&3)
```

قبل هذه التعليمات:
- `cl` يحوي العداد `ecx` (يبدأ من 37 ويتناقص).
- `dl` يحوي `bx & 3`.

بعد `xchg cl, dl` يصبح `cl = bx & 3`، ثم `rol ah, cl` يدوّر `ah` (قيمته `0x01`) بمقدار `bx & 3`:

| `bx & 3` | `ah` بعد التدوير |
|----------|------------------|
| 0 | `0x01` |
| 1 | `0x02` |
| 2 | `0x04` |
| 3 | `0x08` |

أي أن `ah` يصبح **قوة من قوى العدد 2**. سنرمز له بـ **R**.

### 7.3 العملية الكاملة

بعد `popf` نستعيد الأعلام التي حُفظت قبل `xchg` و`rol`، وعلم الحمل `CF = 1` (من `sahf` حيث `ah = 0x01`). فتتحول `adc al, ah` إلى:

```text
al = al + R + 1
```

وبما أن `al` خُصع لـ XOR مع `0xC7`، فإن العملية الكاملة على بايت المدخل هي:

```text
T = ((input_byte XOR 0xC7) + R + 1) mod 256
```

حيث `T` هو البايت المتوقع (المخفي) الذي يجب أن تُقارن به القيمة.

> **مهم:** قيمة `R` تعتمد على `bx`، و`bx` هو المجموع التراكمي لقيم `al` السابقة (أي `T` لكل بايت تمت مقارنته بنجاح). هذا هو **مربط الفرس**: لحساب `input_byte` المطلوب نحتاج إلى معرفة `bx` قبل معالجة هذا البايت.

---

## 8. استراتيجية العكس (Reverse the Loop)

الحلقة تعالج البايتات من **الأخير إلى الأول** في المصفوفة المخفية (بسبب `lea edi, [edi+ecx-1]` ثم `sub edi, 2` مع `scasb` في كل دورة)، بينما تقرأ المدخل بالتسلسل الأمامي. إذاً:

```text
hidden[36]  يُقارن مع  input[0]
hidden[35]  يُقارن مع  input[1]
...
hidden[0]   يُقارن مع  input[36]
```

لعكس العملية نحصل على المدخل بالترتيب الصحيح. المعادلة العكسية:

```text
(input_byte XOR 0xC7) = (T - R - 1) mod 256
input_byte = ((T - R - 1) mod 256) XOR 0xC7
```

خطوات الحل لكل بايت بدءاً من `hidden[36]` وصولاً إلى `hidden[0]`:

1. احسب `R = 1 << (bx & 3)`.
2. احسب `input_byte = ((T - R - 1) & 0xFF) XOR 0xC7`.
3. خزّن النتيجة (ستكون معكوسة الترتيب).
4. حدّث `bx = (bx + T) & 0xFFFF`.

### مثال تطبيقي خطوة بخطوة

البيانات المخفية (37 بايت) كما استخرجناها:

```text
hidden = [
  0xAF, 0xAA, 0xAD, 0xEB, 0xAE, 0xAA, 0xEC, 0xA4, 0xBA, 0xAF, 0xAE, 0xAA,
  0x8A, 0xC0, 0xA7, 0xB0, 0xBC, 0x9A, 0xBA, 0xA5, 0xA5, 0xBA, 0xAF, 0xB8,
  0x9D, 0xB8, 0xF9, 0xAE, 0x9D, 0xAB, 0xB4, 0xBC, 0xB6, 0xB3, 0x90, 0x9A, 0xA8
]
```

**الخطوة 1 (i = 36):** `T = 0xA8`, `bx = 0x0000`

- `bx & 3 = 0` → `R = 1`
- `input_byte = ((0xA8 - 1 - 1) & 0xFF) XOR 0xC7 = 0xA6 XOR 0xC7 = 0x61` → `'a'`
- `bx = 0x0000 + 0xA8 = 0x00A8`

**الخطوة 2 (i = 35):** `T = 0x9A`, `bx = 0x00A8`

- `0xA8 & 3 = 0` → `R = 1`
- `input_byte = ((0x9A - 1 - 1) & 0xFF) XOR 0xC7 = 0x98 XOR 0xC7 = 0x5F` → `'_'`
- `bx = 0x00A8 + 0x9A = 0x0142`

**الخطوة 3 (i = 34):** `T = 0x90`, `bx = 0x0142`

- `0x0142 & 3 = 2` → `R = 4`
- `input_byte = ((0x90 - 4 - 1) & 0xFF) XOR 0xC7 = 0x8B XOR 0xC7 = 0x4C` → `'L'`
- `bx = 0x0142 + 0x90 = 0x01D2`

وهكذا حتى نهاية البايتات.

---

## 9. من أين أتت البيانات المخفية؟

بالانتقال إلى دالة `start`، نرى مجموعة من التعليمات غير المفهومة. غالباً هذه الـ opcodes وُضعت بالصدفة المقصودة، وبالنزول قليلاً نرى مجموعة من البيانات معرّفة في قسم `.text`. لنعد منها **37 بايت** لتكون المصفوفة المخفية التي تُسحب قبل بداية الدالة `sub_401000`:

![البيانات المخفية في IDA](/assets/img/flareon15-2/ida1.png)
_شكل (4): البيانات المخفية داخل قسم .text_

---

## 10. كتابة سكريبت IDC لاستخراج العلم

الآن يمكننا كتابة سكريبت **IDC** (متوفر في النسخة المجانية من IDA) لإيجاد العلم:

```c
#include <idc.idc>

static main()
{
    auto arg0, i, bx, ref_addr, ref_byte, shift, rot, candidate, out, flag;
    arg0 = 0x4010E4;    // عنوان البيانات المخفية
    bx = 0;
    flag = "";          // لتجميع النتيجة كنص

    for (i = 0; i < 37; i++)
    {
        // العنوان المرجعي = arg0 + 36 - i (تقدم خطي عكسي)
        ref_addr = arg0 + 36 - i;
        ref_byte = Byte(ref_addr);
        shift = bx & 3;
        rot = 1 << shift;   // 1, 2, 4, 8

        for (candidate = 0; candidate < 256; candidate++)
        {
            out = ((candidate ^ 0xC7) + rot + 1) & 0xFF;   // لاحظ XOR مع 0xC7
            if (out == ref_byte)
            {
                flag = flag + sprintf("%c", candidate);   // أضف الحرف مباشرة
                bx = (bx + out) & 0xFFFF;
                break;
            }
        }
    }

    Message("Flag: %s\n", flag);
}
```

### كيفية تنفيذ الكود داخل IDA؟

اذهب إلى **File → Script file...** واختر ملف الامتداد `.idc` الذي يحتوي على الكود أعلاه.

---

## 11. النتيجة والتحقق النهائي

عند تنفيذ السكريبت يظهر العلم التالي:

![نتيجة السكريبت](/assets/img/flareon15-2/ida2.png)
_شكل (5): العلم الناتج من السكريبت_

نقوم بنسخه وتجربته:

> **`a_Little_b1t_harder_plez@flare-on.com`**

![تجربة العلم الصحيح](/assets/img/flareon15-2/cmd2.png)
_شكل (6): التحقق من صحة العلم_

وهكذا نكون قد حللنا التحدي الثاني بنجاح، ونستمر إن شاء الله.
