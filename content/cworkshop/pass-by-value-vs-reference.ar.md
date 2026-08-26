---
title: "التمرير بالمرجع والقيمة"
description: "التمرير بالقيمة والمرجع بتتبع كامل في Assembly x86-64 — والقاعدة الذهبية لتمييز mov عن lea."
date: 2026-08-13T10:00:00+03:00
slug: "pass-by-value-vs-reference"
weight: 14
hex: "0x13"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "assembly", "x86-64", "pointers"]
translationKey: "pass-by-value-vs-reference"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

من هذا المقال نبدأ الدخول إلى **Assembly x86-64** لنكتسب فهماً Low-Level. الشرح سيكون بسيطاً وواضحاً.

> **بيئة الأمثلة**: المخرجات هنا من gcc على لينكس (اتفاقية SysV حيث يُمرر المعامل الأول في RDI). على ويندوز x64 تختلف مسجلات المعاملات (RCX/RDX/R8/R9) كما شرحنا في مقال الذاكرة الظاهرية.

> **يُنصح** قبل إكمال هذه السلسلة بدراسة سلسلة الـ Computer Architecture and Organization، ومعرفة أساسية بالـ Memory والـ Registers (شرحناها في مقالي المؤشرات و Stack vs Heap).

## مقدمة إلى الـ Assembly

الـ Assembly لغة برمجة منخفضة المستوى (اختصاراً asm) — سُميت Low-Level لأنها **الأقرب للـ CPU** من غيرها (C/C++، Java...)، وهي **أول طبقة يفهمها الإنسان** من الكود الذي ينفذه المعالج.

```text
C/C++  →  Assembly  →  Machine Code (Hex)  →  Binary  →  CPU
```

كل أمر في الـ asm يسمى **Mnemonic**، يُترجم إلى **Opcode** (تمثيل Hex لتعليمات الآلة)، ثم يتحول إلى Binary ينفذه الـ CPU.

> **ملاحظة**: الفاصلة المنقوطة `;` في الـ asm هي للتعليق (comment).

### أهم الـ Mnemonics (تمثل ~90% من كود الـ asm)

**1. نقل البيانات:**

```asm
mov  destination, source  ; نقل البيانات من المصدر إلى الوجهة (مثل x = 5)
```

**2. العمليات الحسابية:**

```asm
add  dest, src  ; dest = dest + src
sub  dest, src  ; dest = dest - src
inc  operand    ; operand++
dec  operand    ; operand--
```

**3. المقارنة والتحكم:**

```asm
cmp  op1, op2   ; يقارن op1 و op2
jmp  label      ; قفز بلا شرط إلى label
```

**4. إدارة المكدس:**

```asm
push value   ; وضع قيمة على المكدس
pop  value   ; استرجاع قيمة من المكدس
call func    ; استدعاء دالة (يضع عنوان العودة)
ret          ; العودة من الدالة إلى عنوان العودة
```

## تمرير المعاملات في C

يتم تمرير المعاملات للدوال بطريقتين رئيسيتين:
- **Pass by Value** — التمرير بالقيمة.
- **Pass by Reference** — التمرير بالمرجع (عبر المؤشرات).

## 1. التمرير بالقيمة (Pass by Value)

عند تمرير معامل بالقيمة، تنشئ الدالة **نسخة** من المتغير الأصلي — أي تغيير داخل الدالة **لا يؤثر** على الأصل.

```c
#include <stdio.h>

void modify(int x) {
    x = 20; // التعديل يؤثر فقط على النسخة المحلية
    printf("inside function: %d\n", x); // 20
}

int main() {
    int num = 10;
    modify(num); // تمرير بالقيمة
    printf("outside function: %d\n", num); // 10 (لم يتغير)
    return 0;
}
```

### تحليل الذاكرة

1. `num` في `main()` له عنوان افتراضي `0x1000` وقيمته 10.
2. عند استدعاء `modify(num)` تنشأ نسخة جديدة `x` (مثلاً عند `0x2000`) بقيمة 10.
3. التعديل `x = 20` يحدث فقط عند العنوان `0x2000`.
4. المتغير الأصلي `num` يبقى عند `0x1000` دون تغيير.

### كيف نحول C إلى ASM؟

باستخدام gcc:

```bash
gcc -S -masm=intel -fno-asynchronous-unwind-tables -fno-pie -O0 file.c -o file.s
```

> - `-masm=intel`: أسلوب إنتل في الكتابة (الأسهل للقراءة).
> - `-fno-asynchronous-unwind-tables`: يزيل ضجيج `.cfi` directives.
> - `-fno-pie`: يبسط العناوين.
> - `-O0`: بدون تحسينات — أوضح كود.

الكود الناتج:

```asm
.LC0:
    .string    "inside function: %d\n"
    .text
    .globl    modify
    .type    modify, @function
modify:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     DWORD PTR [rbp-4], edi   ; النسخة x = المعامل الأول (edi)
    mov     DWORD PTR [rbp-4], 20    ; x = 20 (النسخة فقط)
    mov     eax, DWORD PTR [rbp-4]
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC0
    mov     eax, 0
    call    printf
    nop
    leave
    ret
    .size    modify, .-modify

    .section    .rodata
.LC1:
    .string    "outside function: %d\n"
    .text
    .globl    main
    .type    main, @function
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     DWORD PTR [rbp-4], 10    ; num = 10
    mov     eax, DWORD PTR [rbp-4]   ; eax = num
    mov     edi, eax                 ; المعامل الأول = 10 (القيمة!)
    call    modify
    mov     eax, DWORD PTR [rbp-4]   ; num لا تزال 10
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC1
    mov     eax, 0
    call    printf
    mov     eax, 0
    leave
    ret
```

### شرح الـ Assembly (تمرير بالقيمة)

**في `main`:**

```asm
main:
    sub     rsp, 16                    ; تخصيص 16 بايت على الستاك
    mov     DWORD PTR -4[rbp], 10      ; [rbp-4] = المتغير num = 10
    mov     eax, DWORD PTR -4[rbp]     ; eax = قيمة num (10)
    mov     edi, eax                   ; نسخ القيمة 10 إلى سجل edi
    call    modify                     ; استدعاء modify
```

**في `modify`:**

```asm
modify:
    mov     DWORD PTR -4[rbp], edi     ; [rbp-4] = edi = 10 (نسخة جديدة، مختلفة عن num)
    mov     DWORD PTR -4[rbp], 20      ; تعديل النسخة إلى 20 — num يبقى دون تغيير
```

**بعد العودة إلى `main`:**

```asm
    mov     eax, DWORD PTR -4[rbp]     ; eax = num (تبقى 10)
```

> **القاعدة الذهبية**: في التمرير بالقيمة نرى `mov` **للقيمة** إلى السجل (`mov edi, eax`) — تُنسخ القيمة نفسها.

## ما هو DWORD PTR؟

مؤشر حجم البيانات — يحدد حجم البيانات التي تتعامل معها التعليمات، لأن **المعالج لا يعرف تلقائياً حجم البيانات** التي يقرأها/يكتبها من الذاكرة.

| المصطلح | الحجم |
|---|---|
| **BYTE** | 1 بايت = 8 بت |
| **WORD** | 2 بايت = 16 بت |
| **DWORD** | 4 بايت = 32 بت |
| **QWORD** | 8 بايت = 64 بت |

**PTR = Pointer** — يشير إلى أننا نتعامل مع عنوان في الذاكرة.

## 2. التمرير بالمرجع (Pass by Reference)

لغة C **لا تحتوي reference variables** مثل C++ — لذلك نستخدم **المؤشرات**. نمرر عنوان المتغير، فتصيب التعديلات الأصل مباشرة.

```c
#include <stdio.h>

void modify(int *x) {
    *x = 20; // التعديل يؤثر على المتغير الأصلي
    printf("inside function:%d\n", *x); // 20
}

int main() {
    int num = 10;
    modify(&num); // تمرير العنوان
    printf("outside function:%d\n", num); // تغير إلى 20
    return 0;
}
```

### تحليل الذاكرة

1. `num` قيمته 10 وعنوانه `0x1000`.
2. عند استدعاء `modify(&num)` يُمرر العنوان `0x1000` إلى المؤشر `x`.
3. التعديل `*x = 20` يغير القيمة عند العنوان `0x1000` مباشرة.
4. المتغير الأصلي `num` يصبح 20.

### الـ Assembly (تمرير بالمرجع)

```asm
.LC0:
    .string    "inside function:%d\n"
    .text
    .globl    modify
    .type    modify, @function
modify:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     QWORD PTR [rbp-8], rdi   ; حفظ العنوان (مؤشر 8 بايت)
    mov     rax, QWORD PTR [rbp-8]   ; rax = العنوان الأصلي
    mov     DWORD PTR [rax], 20      ; *x = 20 (عند العنوان الأصلي مباشرة)
    mov     rax, QWORD PTR [rbp-8]
    mov     eax, DWORD PTR [rax]
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC0
    mov     eax, 0
    call    printf
    nop
    leave
    ret
    .size    modify, .-modify

    .section    .rodata
.LC1:
    .string    "outside function:%d\n"
    .text
    .globl    main
    .type    main, @function
main:
    push    rbp
    mov     rbp, rsp
    sub     rsp, 16
    mov     rax, QWORD PTR fs:40      ; بداية حماية الـ stack canary
    mov     QWORD PTR [rbp-8], rax
    xor     eax, eax
    mov     DWORD PTR [rbp-12], 10    ; num = 10
    lea     rax, [rbp-12]             ; rax = عنوان num (&num)
    mov     rdi, rax                  ; rdi = العنوان (يُمرر كمعامل)
    call    modify
    mov     eax, DWORD PTR [rbp-12]   ; eax = num (أصبحت 20)
    mov     esi, eax
    mov     edi, OFFSET FLAT:.LC1
    mov     eax, 0
    call    printf
    mov     eax, 0
    mov     rdx, QWORD PTR [rbp-8]    ; التحقق من الـ canary
    sub     rdx, QWORD PTR fs:40
    je      .L4
    call    __stack_chk_fail          ; إذا تعدل الـ canary ← فشل
.L4:
    leave
    ret
```

### شرح الـ Assembly (تمرير بالمرجع)

**في `main`:**

```asm
    mov     DWORD PTR -12[rbp], 10   ; [rbp-12] = المتغير num = 10
    lea     rax, -12[rbp]            ; rax = عنوان num (&num)
    mov     rdi, rax                 ; rdi = العنوان (يمرر كمعامل)
    call    modify
```

**في `modify`:**

```asm
    mov     QWORD PTR -8[rbp], rdi   ; يحفظ العنوان الأصلي في متغير محلي
    mov     rax, QWORD PTR -8[rbp]   ; rax = العنوان الأصلي
    mov     DWORD PTR [rax], 20      ; *rax = 20 — عند العنوان الأصلي مباشرة
```

**بعد العودة إلى `main`:**

```asm
    mov     eax, DWORD PTR -12[rbp]  ; eax = num (20) — القيمة الجديدة
```

> **القاعدة الذهبية**: في التمرير بالمرجع نرى `lea` **لحساب العنوان** ثم `mov` إلى سجل 64-bit (`mov rdi, rax`) — يُمرر **العنوان** لا القيمة.

## الأمر الجديد: `lea`

**Load Effective Address** — يحسب العنوان الفعلي (Effective Address) لتعريف الذاكرة ويخزنه في الوجهة **دون الوصول إلى الذاكرة**.

```asm
lea rax, [rbp-12]   ; rax = &num — حساب العنوان فقط، لا قراءة من الذاكرة
```

> **الفرق الجوهري**: `mov eax, [rbp-4]` تقرأ **القيمة** من الذاكرة، بينما `lea rax, [rbp-12]` تحسب **العنوان** فقط. هذا هو السبب في أن رؤية `lea` في الـ disassembly تخبرك أن الكود كان يمرر بالمرجع.

## حالة خاصة: تمرير المصفوفات

المصفوفات تُمرر **تلقائياً بالمرجع** — لأن اسم المصفوفة يتحول إلى مؤشر لأول عنصر.

```c
#include <stdio.h>

void changeArray(int arr[]) {
    arr[0] = 100; // تعديل مباشر على المصفوفة الأصلية
}

int main() {
    int a[3] = {1, 2, 3};
    changeArray(a); // تمرير العنوان بدون &
    printf("%d\n", a[0]); // 100
    return 0;
}
```

## ملاحظة مهمة: المؤشر نفسه نسخة

عند تمرير مؤشر، تُمرر **نسخة من العنوان** — يمكنك تعديل القيمة التي يشير إليها (`*ptr = value`)، لكن إن غيّرت المؤشر نفسه (`ptr = &new`) لن يتأثر المؤشر الأصلي خارج الدالة. لتغيير المؤشر الأصلي نستخدم **مؤشر إلى مؤشر** (`int **ptr`):

```c
#include <stdio.h>
#include <stdlib.h>

void changePointer(int **ptrToPtr) {
    int *new_ptr = malloc(sizeof(int));
    *new_ptr = 100;
    *ptrToPtr = new_ptr; // تغيير المؤشر الأصلي ليشير إلى الجديد
}

int main() {
    int x = 5;
    int *org_ptr = &x;

    printf("pointer before change: %d\n", *org_ptr); // 5

    changePointer(&org_ptr); // تمرير عنوان المؤشر

    printf("pointer after change: %d\n", *org_ptr); // 100

    free(org_ptr);
    return 0;
}
```

## أسئلة مهمة (وأجوبتها)

**1. كيف نعرف أن التمرير بالمرجع؟**
بوجود الأمر `lea` — لأنه يحسب العناوين دون التدخل في الذاكرة. نمط `lea` + `mov rdi, rax` = تمرير بالمرجع.

**2. ليش استخدمنا `rax` مع by-reference و `eax` مع by-value؟**
`eax` هو النصف السفلي 32-bit من `rax`. مع العناوين/المؤشرات نستخدم سجلات 64-bit كاملة لأن **العنوان 8 بايت** — استخدام 32-bit يقسم العنوان (4 بايت فقط).
> تفصيلة إضافية: في x64، الكتابة إلى سجل 32-bit (مثل `mov eax, ...`) **تصفّر النصف العلوي** من سجل 64-bit تلقائياً — وهذا سلوك مهم عند تحليل الكود.

**3. ليش `[rbp-12]` في by-reference و `[rbp-4]` في by-value؟**
لأن مثال by-reference يحتوي **Stack Canary** — قيمة عشوائية (8 بايت) توضع بين المتغيرات المحلية وعنوان العودة لحمايته من هجمات Buffer Overflow:
- GCC يضيف الـ canary تلقائياً عندما **يُمرَّر عنوان متغير محلي إلى دالة أخرى** (وهذا بالضبط ما يحدث في by-reference).
- عند نهاية الدالة يُتحقق من الـ canary؛ إن تعدّل (تجاوز buffer) يستدعي `__stack_chk_fail` ويوقف البرنامج.

تخطيط الستاك:

```text
by value:                 by reference:
+-----------------+       +-----------------+
|  num  [rbp-4]   |       |  num  [rbp-12]  |
+-----------------+       |  canary [rbp-8] |
| return address  |       +-----------------+
+-----------------+       | return address  |
                          +-----------------+
```

## لماذا أدخلنا الـ Assembly في شرح لغة C؟

لتصل إلى الأمور المتقدمة في الـ Reverse Engineering يجب أن **تفهم الكود سطراً سطراً في الـ Low-Level**. التعلم العملي مع رؤية كيف يتحول كود C إلى Assembly — وفهم كيف يمشي الـ asm مع الـ C — هو الطريقة الأكثر فعالية للبدء.

## ربط الـ RE

1. **قاعدة `lea` = by-reference**: عند تحليل أي binary، تتبع أنماط `lea` تكشف لك المعاملات الممررة بالمرجع — وهذا يساعد على فهم معنى الباراميترات في دوال النظام والـ APIs.
2. **المعامل الأول في RDI/EDI** (SysV x86-64): قاعدة ثابتة — أول معامل دائماً في RDI/EDI. معرفتها تجعل قراءة أي disassembly أسرع بكثير. أما في ويندوز x64 فالمعامل الأول في RCX — راجع اتفاقية x64 في مقال الذاكرة الظاهرية.
3. **Stack Canary**: ظهور أنماط `fs:40` في الـ disassembly يعني أن الكود مترجم مع حماية ضد Buffer Overflow — وهذا يحدد مسبقاً نوع التحدي الذي سيواجهه المستغل.

> حتى تقرأ الكود بالـ Low-Level، اكتب برامج C صغيرة وراجع الـ `gcc -S` الناتج — إنها أسرع طريقة لبناء الحدس. والسلام عليكم ورحمة الله وبركاته.

