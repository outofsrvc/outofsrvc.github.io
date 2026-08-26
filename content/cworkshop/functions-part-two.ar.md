---
title: "الدوال — الجزء الثاني"
description: "نطاق المتغيرات وملفات الرأس وstatic والعودية — بتتبع كامل لإطار المكدس خطوة بخطوة."
date: 2026-08-13T10:00:00+03:00
slug: "functions-part-two"
weight: 15
hex: "0x14"
stage: "advanced"
categories: [c-lang]
tags: ["c-lang", "assembly", "functions", "recursion", "static"]
translationKey: "functions-part-two"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

بعد أن بنينا حجر الأساس في لغة C وتقسيم الذاكرة والمؤشرات ومقدمة الـ Assembly — نكمل الطريق نحو بنية معرفية كاملة. في هذا المقال نتحدث عن الدوال بطريقة **متقدمة**، بمرجع أساسي من كتاب K&R، مع تتبع السجلات في الـ Assembly لنرى كيف يمشي الكود في الـ Low-Level.

**فهرس المقال:**
- 1.1 نطاق المتغيرات (Variables Scope)
- 1.2 ملفات الرأس (Header Files)
- 1.3 المتغيرات والدوال الثابتة (Static)
- 1.4 العودية (Recursion)

## المكونات الأساسية للدوال

| المكوّن | الوصف |
|---|---|
| **التوقيع (Signature)** | تعريف الدالة: `returnType funcName(Parameters) {}` |
| **الجسم (Body)** | الوظيفة التي تقوم بها الدالة |
| **الباراميترات** | قيم الإدخال |
| **قيمة الإرجاع** | الناتج الرئيسي |

## نموذج تنفيذ دالة

```c
int add(int a, int b) {
    int result = a + b;
    return result;
}
```

### الـ Assembly (32-bit)

نستخدم `-m32` لتبسيط الشرح (نمط EBP الكلاسيكي الذي تراه في ملايين البرامج القديمة):

```bash
gcc -S -masm=intel -m32 -fno-asynchronous-unwind-tables -fno-pie -O0 file.c -o file.s
```

> على Arch Linux: `sudo pacman -S gcc-multilib lib32-gcc-libs lib32-glibc`
> على Kali/Debian: `sudo apt install gcc-multilib libc6-dev-i386 libc6:i386`

```asm
add:
    push    ebp
    mov     ebp, esp
    sub     esp, 16
    mov     edx, DWORD PTR [ebp+8]   ; edx = a = 5 (أول باراميتر)
    mov     eax, DWORD PTR [ebp+12]  ; eax = b = 3 (ثاني باراميتر)
    add     eax, edx                 ; eax = 3 + 5 = 8
    mov     DWORD PTR [ebp-4], eax   ; result = 8
    mov     eax, DWORD PTR [ebp-4]   ; eax = result (قيمة الإرجاع)
    leave                            ; mov esp, ebp + pop ebp
    ret
main:
    push    ebp
    mov     ebp, esp
    push    3                        ; يمرر b = 3 أولاً (يمين-ليسار)
    push    5                        ; ثم a = 5
    call    add
    add     esp, 8                   ; تنظيف المعاملات — المتصل (cdecl)
    mov     eax, 0
    leave
    ret
```

### شرح خطوة بخطوة

**1. إنشاء إطار المكدس لـ `add`:**

```asm
add:
    push ebp      ; حفظ EBP الحالي (خاص بـ main)
    mov  ebp, esp ; إنشاء إطار جديد — أول تعليمتين في معظم الدوال
```

**2. تخصيص مساحة للمتغيرات المحلية:**

```asm
sub esp, 16      ; 16 بايت (المتغير result 4 بايت + الباقي محاذاة/padding)
```

**3. قراءة المدخلات وإجراء العمليات:**

```asm
mov edx, DWORD PTR [ebp+8]   ; edx = 5 (أول باراميتر a)
mov eax, DWORD PTR [ebp+12]  ; eax = 3 (ثاني باراميتر b)
add eax, edx                 ; eax = 3 + 5 = 8
```

**4. حفظ الناتج وإرجاعه:**

```asm
mov DWORD PTR [ebp-4], eax   ; تخزين الناتج في المتغير المحلي
mov eax, DWORD PTR [ebp-4]   ; حمل الناتج إلى eax (قيمة الإرجاع)
leave                        ; mov esp, ebp + pop ebp
ret                          ; إخراج عنوان الرجوع والعودة لـ main
```

### تتبع المؤشر ESP (خريطة الـ Stack)

**1. بدء `main` (بعد `mov ebp, esp`):**

```text
[ret address]   : عنوان الرجوع للنظام
[saved ebp]     : ebp/esp الحالي
```

**2. بعد `push 3` + `push 5`:**

```text
[ret address]
[saved ebp]
3
5            ← esp هنا
```

**3. بعد `call add` (يضع عنوان الرجوع):**

```text
[ret address]
[saved ebp]
3
5
[ret address]   : عنوان الرجوع لـ main
```

**4. داخل `add` بعد `mov ebp, esp`:**

```text
[ret address]
[saved ebp]
3
5
[ret address]
[saved new ebp]  : ebp/esp الخاص بـ add
```

**5. بعد `sub esp, 16`:**

```text
[ret address]
[saved ebp]
3               : [ebp + 12]  → b
5               : [ebp + 8]   → a
[ret address]   : [ebp + 4]
[saved new ebp] : [ebp]
[result]        : [ebp - 4]
[padding]       : [esp]
```

**6. بعد `leave`** — يعيد esp إلى موضع saved ebp، ثم يستعيده.
**7. بعد `ret`** — يقفز إلى عنوان الرجوع في main.
**8. بعد `add esp, 8`** — ينظف المعاملين 3 و 5.

> **لاحظ**: `add esp, 8` بعد الـ call تعني أن هذه هي اتفاقية **cdecl** (المتصل ينظف). لو كانت stdcall لرأيت `ret 8` داخل الدالة.

## 1.1 نطاق المتغيرات (Variables Scope)

النطاق يحدد المكان الذي يمكن أن يصل إليه المتغير في الكود.

### 1. النطاق المحلي (Block Scope)

**المكان**: داخل الكتلة `{}` (دالة، حلقة، شرط...).
**الوصول**: مرئي داخل الكتلة فقط، غير مرئي خارجها.

```c
void example() {
    int x = 10;          // محلي داخل الدالة
    if (x > 5) {
        int y = 20;      // محلي داخل if
        printf("%d", y); // يطبع
    }
    printf("%d", y);     // خطأ — y غير مرئي هنا
}
```

### 2. النطاق العام (File Scope)

**المكان**: خارج أي دالة (على مستوى الملف).
**الوصول**: يمكن الوصول إليه من أي مكان في الملف.

```c
int global = 50;

void func1() { global++; }      // مسموح
void func2() { printf("%d", global); } // يطبع 51
```

### 3. نطاق المعاملات (Function Arguments Scope)

**المكان**: داخل أقواس `()` الدالة.

```c
int sum(int a, int b) {  // a, b داخل هذه الدالة فقط
    return a + b;
}
```

### 4. ظاهرة الإخفاء (Shadowing)

عند تعريف متغير محلي بنفس اسم متغير عام، يُحجب المتغير العام وتُستخدم النسخة المحلية.

```c
int x = 100; // متغير عام

void demo() {
    int x = 200;     // يخفي المتغير العام
    printf("%d", x); // يطبع 200
}
```

### ملاحظات مهمة

1. المتغيرات تعرّف من بداية الكتلة `{}`.
2. المتغيرات العامة تُعرّف خارج الدوال.
3. النطاق يبدأ من مكان التعريف حتى نهاية الكتلة (local) أو نهاية الملف (global).
4. الكتل المتداخلة تصل إلى متغيرات الكتل الخارجية إذا لم يحدث shadowing.

## 1.2 ملفات الرأس (Header Files)

ملفات `.h` (ملفات الـ include) تحتوي إعلانات (Declarations) تُشارك بين عدة ملفات مصدرية `.c` — لتجنب تكرار الإعلانات، وفصل الواجهة (Interface) عن التنفيذ (Implementation).

### محتويات ملفات الرأس

1. **إعلانات الدوال** (بدون الجسم):

```c
int add(int a, int b);
```

2. **المتغيرات الخارجية** (extern):

```c
extern int counter;
```

3. **الماكروز**:

```c
#define MAX_SIZE 100
```

4. **تعريفات الهياكل والاتحادات** (structs/unions — قادمة).
5. **تعريفات النماذج** (typedef — قادمة).
6. **تضمينات مكتبات النظام**:

```c
#include <stdio.h>
```

### مثال عملي (فصل الواجهة عن التنفيذ)

**calc.h** — ملف الرأس:

```c
#ifndef CALC_H
#define CALC_H

// إعلان الدالة
int add(int a, int b);

// إعلان متغير خارجي
extern int result;

#endif
```

**main.c** — الاستخدام:

```c
#include "calc.h"

int result = 0; // التعريف الفعلي (يوجد في ملف واحد فقط)

int main() {
    result = add(5, 3);
    return 0;
}
```

**math.c** — التنفيذ:

```c
#include "calc.h"

int add(int a, int b) {
    return a + b;
}
```

> **إعلان (Declaration) vs تعريف (Definition)**: الـ extern مجرد إعلان "يوجد متغير بهذا الاسم في مكان ما" — أما التعريف فهو الذي يخصص الذاكرة فعلياً (`int result = 0;`) ويجب أن يظهر في ملف واحد فقط. هذا التمييز يهمك في الـ RE عند تمييز الـ symbols المعلنة من المعرفة.

### حماية التضمين المتكرر (Include Guards)

تمنع تكرار التضمين إذا ضُمّن الملف أكثر من مرة (تمنع أخطاء التضارب):

```c
#ifndef MY_HEADER_H  // إذا لم يُعرف هذا الرمز
#define MY_HEADER_H  // عرّفه الآن
// محتويات الملف
#endif
```

### تحذيرات هامة

- **لا** تضع تعريفات في ملفات الرأس (مثل `int x;`).
- استخدم `extern` للمتغيرات المشتركة.
- **لا** تضع جسم الدوال في الرأس — بل في ملف التنفيذ.

## 1.3 المتغيرات والدوال الثابتة (Static)

تُستخدم `static` في ثلاثة سياقات:

### 1. متغير ثابت في نطاق الملف

نطاق الرؤية محصور بالملف الحالي فقط — لا يمكن الوصول له من ملفات أخرى حتى مع `extern`:

```c
// in file1.c
static int internal_var; // غير مرئي خارج الملف

void func() {
    internal_var = 42;
}
```

### 2. متغير ثابت داخل دالة

يحتفظ بقيمته **بين الاستدعاءات** — يُهيأ مرة واحدة فقط (أول استدعاء) ويبقى طوال عمر البرنامج:

```c
void counter() {
    static int count = 0; // تهيئة مرة واحدة فقط
    count++;
    printf("Count: %d\n", count);
}
```

الاستدعاء الأول → `count = 1`، الثاني → `count = 2`.

### 3. دالة ثابتة (static function)

مرئية فقط داخل الملف الذي عُرّفت فيه — لا يمكن استدعاؤها من ملفات أخرى:

```c
static void helper() {
    // محلية لهذا الملف فقط
}
```

### التهيئة التلقائية والتخزين

المتغيرات الثابتة غير المهيأة تُهيأ تلقائياً إلى صفر:

```c
static int x;  // x = 0
static int *p; // p = NULL
```

> **موقع التخزين**:
> - static **مهيأ** بقيمة → `.data` segment.
> - static **غير مهيأ** (يُصفَّر) → `.bss` segment.
> هذا يطابق ما شرحناه في تخطيط الذاكرة — ويميز عنواناً ثابتاً في الـ binary وليس إزاحة عن EBP.

### جدول المقارنة

| | **static** | **extern** |
|---|---|---|
| الرؤية | الملف الحالي فقط | كل الملفات |
| التخزين | `.data`/`.bss` | `.data`/`.bss` |
| التهيئة | تلقائي للصفر إن لم تُهيأ | حسب التعريف |
| الهدف | إخفاء التفاصيل | المشاركة بين الملفات |

## 1.4 العودية (Recursion)

أسلوب تستدعي فيه الدالة **نفسها** لحل المشكلة — تقسّم المشكلة الرئيسية إلى مسائل فرعية أصغر من نفس النوع حتى تصبح بسيطة قابلة للحل المباشر.

تعتمد على مبدأين:
1. **الحالة الأساسية (Base Case)**: حالة توقف العودية — أبسط حالة تُحل مباشرة (`0! = 1`).
2. **الخطوة التكرارية (Recursive Step)**: تقسيم المشكلة إلى جزء بسيط + استدعاء أصغر لنفس الدالة.

### مثال: حساب العاملي (Factorial)

```c
#include <stdio.h>

int factorial(int n) {
    if (n == 0)              // الحالة الأساسية
        return 1;
    else                     // الخطوة التكرارية
        return n * factorial(n-1); // عودية
}

int main() {
    printf("%d! = %d\n", 3, factorial(3)); // يطبع 6
    return 0;
}
```

### خطوة بخطوة

```text
1. factorial(3): 3 != 0 → 3 * factorial(2)
2. factorial(2): 2 != 0 → 2 * factorial(1)
3. factorial(1): 1 != 0 → 1 * factorial(0)
4. factorial(0): 0 == 0 → 1

الرجوع: 1*1 = 1 → 1*2 = 2 → 2*3 = 6
```

## ربط الـ RE — لماذا هذا المقال أساسي

1. **`[ebp+4]` = Return Address = هدف الاستغلال**: خريطة الـ stack التي رسمناها هي بالضبط ما يستغله الـ **Buffer Overflow** — المدخل المتجاوز يكتب فوق المتغيرات المحلية ← saved EBP ← **عنوان العودة**، فيعيد توجيه التنفيذ إلى shellcode أو ROP chain. هذا المقال يبني فهم البنية التي ستشرحها في الـ exploitation.

2. **العودية في الـ assembly = تكدس إطارات**: كل استدعاء ذاتي يضيف إطاراً جديداً (call/ret + prologue). غياب base case = **Stack Overflow** (نفاد المكدس) — وهو أيضاً أسلوب هجمات DoS. عند تحليل binary، العودية تظهر كأنماط call متكررة للعنوان نفسه.

3. **المتغيرات الـ static = عنوان ثابت**: في الـ disassembly تظهر كـ **عنوان مطلق في `.data`/`.bss`** وليس كإزاحة عن EBP — تمييز بصري سريع في IDA يكشف لك المتغيرات العامة فوراً.

4. **Shadowing لا يُرى في الـ asm**: يحل وقت الترجمة — المترجم قد يعيد استخدام نفس الإزاحة. لن ترى له أثراً في الكود المترجم.

5. **الـ header files وأثرها في الـ binary**: إعلان vs تعريف يظهر في الـ symbols — وفصل التنفيذ عن الواجهة يعني أن الدوال المشتركة تظهر كـ imports/exports في الـ PE.

---

إن شاء الله تكون هذه المفاهيم واضحة، وستكون معنا في كل برنامج نكودّه. أعتذر إن كانت المقالات طويلة — ولكن هذا حق العلم الذي يجب أن نوفيه حقه.

