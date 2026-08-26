---
title: "الجمل الشرطية"
description: "if وelse وswitch والعامل الثلاثي — وكيف تتحول إلى cmp وقفزات شرطية وجداول قفزات في الـ disassembly."
date: 2026-08-13T10:00:00+03:00
slug: "conditionals"
weight: 6
hex: "0x05"
stage: "basics"
categories: [c-lang]
tags: ["c-lang", "basics", "assembly"]
translationKey: "conditionals"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

في هذا المقال سنتعرف على if / else-if / else، الجمل المتداخلة (Nested if)، جملة switch، والعامل الثلاثي (Ternary Operator).

**الجمل الشرطية**: تُستخدم لاتخاذ القرارات بناءً على شروط معينة — تُنفَّذ تعليمات مختلفة بناءً على نتيجة الشرط (true أو false).

## 1. جملة if

تُستخدم للتحقق من شرط معين وتنفيذ الكود إذا كان الشرط صحيحاً (true). وهي أبسط الجمل.

```c
if (condition) {
    // كود يُنفذ إذا كان الشرط صحيحًا
}
```

## 2. جملة if-else

تُنفذ كوداً معيناً إذا كان الشرط صحيحاً، وكوداً مختلفاً إذا كان خاطئاً.

```c
if (condition) {
    // كود يُنفذ إذا كان الشرط صحيحًا
} else {
    // كود يُنفذ إذا كان الشرط خاطئًا
}
```

## 3. جملة if-else-if-else

تُستخدم عندما يكون هناك أكثر من شرط. يتم التحقق من الشروط بالتسلسل حتى يُصبح أحدها صحيحاً.

```c
if (condition1) {
    // كود يُنفذ إذا كان الشرط 1 صحيحًا
} else if (condition2) {
    // كود يُنفذ إذا كان الشرط 2 صحيحًا
} else {
    // كود يُنفذ إذا كانت كل الشروط خاطئة
}
```

### مثال

```c
#include <stdio.h>

int main() {
    int grade = 85;

    if (grade >= 90) {
        printf("Excellent\n");
    } else if (grade >= 75) {
        printf("Good\n");
    } else {
        printf("Needs Improvement\n");
    }

    return 0;
}
```

## 4. الشروط المركبة (`&&` و `||`)

يمكن دمج أكثر من شرط في جملة if واحدة:

```c
if (age >= 18 && hasID) {
    // ينفذ فقط إذا تحقق الشرطان معاً (AND)
}

if (error == 0 || retry < 3) {
    // ينفذ إذا تحقق أحدهما على الأقل (OR)
}
```

## 5. الجمل المتداخلة (Nested if)

كتابة جملة if داخل جملة if أخرى للتحقق من شروط متعددة متتالية.

```c
if (condition1) {
    if (condition2) {
        // كود يُنفذ إذا كان الشرطان صحيحين
    }
}
```

### مثال

```c
#include <stdio.h>

int main() {
    int age = 20;
    int hasID = 1;

    if (age >= 18) {
        if (hasID) {
            printf("You can enter.\n");
        } else {
            printf("You need an ID.\n");
        }
    } else {
        printf("You are underage.\n");
    }

    return 0;
}
```

## 6. جملة switch-case

نستخدمها لاختبار قيمة متغير معين مع لائحة من الاحتمالات نضعها نحن. إذا تساوت القيمة مع أي احتمال، ستُنفذ الأوامر الموضوعة في ذلك الاحتمال. كل احتمال يسمى `case`.

```c
switch (variable) {
    case value1:
        // كود يُنفذ إذا كانت قيمة المتغير value1
        break;
    case value2:
        // كود يُنفذ إذا كانت قيمة المتغير value2
        break;
    default:
        // كود يُنفذ إذا لم تتطابق القيمة مع أي حالة
}
```

### مثال

```c
#include <stdio.h>

int main() {
    int day = 3;

    switch (day) {
        case 1:
            printf("Monday\n");
            break;
        case 2:
            printf("Tuesday\n");
            break;
        case 3:
            printf("Wednesday\n");
            break;
        default:
            printf("Invalid day\n");
    }

    return 0;
}
```

> **يجب استخدام `break` لكل case** — لأنها إن لم تُستخدم، سيستمر التنفيذ في الحالات التالية بغض النظر عن القيمة (وهذا يسمى fall-through). جرب إزالة `break` من المثال وسترى السلوك بنفسك.

> **ملاحظة هامة**: الشروط في if أو switch تعتمد على القيم المنطقية:
> - `0` = خاطئ (false)
> - غير `0` = صحيح (true)

## 7. العامل الثلاثي (Ternary Operator)

اختصار لجملة if-else البسيطة.

```c
condition ? true_expression : false_expression
```

### مثال

```c
int a = 10, b = 20;
int max = (a > b) ? a : b;   // نفس معنى if-else أدناه
```

هذا يساوي تماماً:

```c
int max;
if (a > b) {
    max = a;
} else {
    max = b;
}
```

## كيف ستبدو هذه الجمل في الـ Assembly؟

هذا هو أهم قسم في المقال — لأن كل ما سبق سيظهر لك في الـ disassembly بهذه الأشكال:

### if → `cmp` + قفزة شرطية

كل جملة if تتحول إلى `cmp` تليها قفزة شرطية (`jz`, `jnz`, `jg`, `jl`...). "الشرط صحيح/خاطئ" هو تحديداً قيمة الـ flags (`ZF`, `SF`, `OF`) التي تحددها المقارنة — وهي نفس قواعد "0 = false، غير 0 = true" التي ذكرناها.

```asm
cmp [grade], 90     ; grade >= 90 ?
jl  not_excellent   ; إن لم يكن أكبر أو يساوي، اقفز
; ... فرع Excellent
```

### switch → جدول القفزات (Jump Table)

عندما يكتب المبرمج switch بعدة حالات متتالية، قد يحوّلها المترجم إلى **جدول قفزات (Jump Table)** — مصفوفة من العناوين مخزنة في قسم `.rdata`، ويصل إليها بفهرس مباشر.

> **مهارة RE أساسية**: عند رؤية jump table في الـ disassembly، تعرف فوراً أن الكود الأصلي كان switch، وتستخرج كل الحالات من الجدول نفسه. أما if-else chains فتظهر كسلسلة مقارنات متتالية — وهذا فارق بصري حاسم في التحليل.

### break في switch → `jmp`

أمر `break` يتحول إلى قفزة غير شرطية للخروج من الـ switch — لهذا يبدو شكل الـ switch في الـ assembly كعقد اللؤلؤ (مقارنات ← قفزات داخلية ← خروج واحد).

### الـ Ternary → لا فرق

المترجم يعامل الـ ternary كـ if-else عادي تماماً — لن ترى له شكلاً مميزاً في الـ assembly.

## تمرين عملي

1. اكتب مثال `grade` وقارنه بمثال `switch` — ثم افحصهما بـ `gcc -S` ولاحظ: الأول ينتج سلسلة `cmp/jl`، والثاني قد ينتج jump table (خذه لاحقاً إلى IDA/Ghidra لرؤية الجدول).
2. جرّب `break` محذوفة من switch وتأكد من سلوك fall-through.
3. أعد كتابة مثال الـ ternary كـ if-else والعكس، وقارن الـ assembly الناتج — ستجدهما متطابقين.
