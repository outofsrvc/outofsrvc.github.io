---
title: "التخصيص الديناميكي"
description: "malloc وcalloc وrealloc وإدارة Bins وtcache — وثغرات سوء التخصيص: Heap Overflow وUAF وDouble Free."
date: 2026-08-13T10:00:00+03:00
slug: "dynamic-allocation"
weight: 13
hex: "0x12"
stage: "memory"
categories: [c-lang]
tags: ["c-lang", "heap", "memory", "malloc", "exploitation"]
translationKey: "dynamic-allocation"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

بعد أن تحدثنا عن الـ Heap والـ Stack وفهمنا البنية الداخلية لكل منهما وآلية عملهما، سنتطرق اليوم لشرح موضوع **تخصيص الذاكرة الديناميكية** بطريقة موسعة — إدارة الذواكر الحرة وأشهر الثغرات المتكررة في هذا الموضوع.

> **Dynamic Memory Allocation**: فهمه مهم جداً لتحليل البرمجيات واستغلال الثغرات وفهم آلية عمل البرنامج من الداخل.

## دوال التخصيص الأساسية

### 1. malloc() و free()

**malloc** تخصص كتلة ذاكرة في الـ Heap بحجم محدد (بالبايت) **دون تهيئة** — محتواها قيم عشوائية (garbage):

```c
void *malloc(size_t size);
```

**free** تحرر الذاكرة المخصصة سابقاً:

```c
void free(void *ptr);
```

### البنية الداخلية للكتلة (malloc_chunk)

كما في المقال السابق، الكتلة في منظم الذاكرة (مثل glibc) لها هيكل:

```c
struct malloc_chunk {
    size_t prev_size;   // حجم الكتلة السابقة (إذا كانت حرة)
    size_t size;        // حجم الكتلة الحالية + Flags
    struct malloc_chunk *fd;  // المؤشر للكتلة الحرة التالية (للكتل الحرة)
    struct malloc_chunk *bk;  // المؤشر للكتلة الحرة السابقة
};
```

- **size**: حجم الكتلة الحالية مع الهيدر، والبتات الثلاث على اليمين (LSB) تستخدم للـ Flags.
- **prev_size**: يُستخدم فقط إذا كانت الكتلة السابقة حرة (PREV_INUSE = 0)، ويتيح **الدمج (Coalescing)** عند التحرير لتقليل التجزئة (Fragmentation).

### آلية malloc النظرية

1. تطلب كتلة بحجم معين.
2. تبحث في الـ Bins (قوائم الكتل الحرة).
3. إذا لم توجد كتلة مناسبة، تطلب ذاكرة من النظام عبر `sbrk()` أو `mmap()`.
4. إذا كانت الكتلة أكبر من المطلوب، تُقسم وتُعاد بقيّة الحجم.

### مثال

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    // تخصيص ذاكرة لعشرة أعداد صحيحة = 40 بايت
    int *nums = (int*)malloc(10 * sizeof(int));

    // فحص نجاح التخصيص
    if (nums == NULL) {
        printf("failed..!\n");
        return 1;
    }

    // استخدام الذاكرة المخصصة
    for (int i = 0; i < 10; i++) {
        nums[i] = i * 10;
    }

    // تحرير الذاكرة
    free(nums);
    return 0;
}
```

تخطيط الذاكرة:

```text
0x1000: [Flags + حجم الكتلة]   -> metadata (مخفي عن المستخدم)
0x1008: [nums[0]]              -> المؤشر nums يشير إلى هنا
0x100C: [nums[1]]
  ~ ~ ~
0x1028: [nums[9]]
```

### آلية free النظرية

1. تتحقق من صحة المؤشر (هل يشير إلى بداية كتلة؟).
2. إذا كانت الكتلة المجاورة حرة، تدمجها مع الكتلة الحالية.
3. تعيد الكتلة إلى إدارة الكتل المحررة (Bins).

> **ركّز على `free` — لأنها ما يخرب الدنيا.**

### أخطاء شائعة يجب تجنبها

1. أوعك تحرر مؤشراً لم تخصصه بإحدى دوال التخصيص.
2. أوعك تحرر نفس الكتلة مرتين (Double Free).
3. أوعك تستخدم الكتلة بعد تحريرها (UAF).
4. بعد التحرير **عيّن المؤشر إلى NULL فوراً**.

## 2. calloc()

تخصص كتلة لمصفوفة من العناصر **وتهيئها بأصفار**:

```c
void *calloc(size_t num, size_t size);
```

- تخصص كتلة تكفي لـ `num` عناصر بحجم `size` لكل عنصر.
- تهيء جميع البايتات بالأصفار (الفرق الجوهري عن `malloc`).
- الحساب الكلي: `num * size`.

تتم التهيئة عبر `memset`:

```c
memset(ptr, 0, num * size);
```

> `memset` تمليء كتلة من الذاكرة بقيمة معينة: `ptr` = المؤشر، القيمة المراد وضعها، وعدد البايتات.

> **ملاحظة هامة**: بسبب التهيئة، `calloc(4, 5)` **لا تساوي** `malloc(20)` — الأولى فيها 20 بايتاً كلها أصفار، والثانية بقيم عشوائية (garbage). هذا يجعل calloc أبطأ لكن أكثر أماناً.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    // تخصيص مصفوفة من 5 عناصر من نوع float
    float *temps = (float*)calloc(5, sizeof(float));

    if (temps == NULL) {
        printf("Failed..!\n");
        return 1;
    }

    // التأكد من التهيئة بصفر
    for (int i = 0; i < 5; i++) {
        printf("temps[%d] = %f\n", i, temps[i]); // سيطبع الكل 0.0
    }

    free(temps);
    return 0;
}
```

## 3. realloc()

تغير حجم الكتلة المخصصة — ولها سيناريوهان:

1. **التوسيع في نفس المكان**: إذا كانت المساحة بعد الكتلة كافية (الكتلة التالية حرة)، تُوسَّع الكتلة في مكانها — والمؤشر الجديد هو نفسه القديم.
2. **النقل**: إذا لم تكن مساحة كافية، تخصص كتلة جديدة وتنقل البيانات عبر `memcpy()`، ثم تحرر القديمة وتعيد المؤشر الجديد.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    char *buffer = (char*)malloc(10);

    if (buffer == NULL) return 1;

    strcpy(buffer, "Hello");

    // توسيع الذاكرة لاستيعاب نص أطول
    char *new_buffer = (char*)realloc(buffer, 20);

    if (new_buffer == NULL) {
        free(buffer); // الحفاظ على الكتلة الأصلية
        return 1;
    }

    buffer = new_buffer; // التحديث بعد التأكد
    strcat(buffer, "World"); // الآن المساحة كافية

    free(buffer);
    return 0;
}
```

> **نمط صحيح**: لا تكتب `buffer = realloc(buffer, ...)` مباشرة — إن فشل realloc ستخسر المؤشر الأصلي. احفظه في مؤشر مؤقت ثم تحقق من NULL قبل التحديث.

## 4. alloca()

لم نتوسع فيها لأنها تخصص على الـ **Stack** وخطورة استخدامها عالية (تسبب ثغرات overflow). نقطة مهمة: ذاكرتها تُحرر تلقائياً عند الخروج من الدالة **بدون free()** — مثل الستاك تماماً.

## إدارة الكتل المحررة (Bins)

عند تحرير كتلة بـ `free()`، لا تُعاد للنظام مباشرة بل تُحفظ في قوائم منظمة تسمى **Bins** لإعادة استخدامها لاحقاً — يحسن الأداء ويقلل التجزئة.

### أنواع الـ Bins الرئيسية

| النوع | الحجم | القوائم | النمط | الدمج |
|---|---|---|---|---|
| **Fast Bins** | صغير (16-88 بايت chunk) | 10 قوائم فردية | LIFO | لا (للسرعة) |
| **Small Bins** | متوسط (32-1024) | 62 قائمة | FIFO | نعم |
| **Large Bins** | كبير (>1024) | 63 قائمة | مرتبة بالحجم (Best-fit) | نعم |
| **Unsorted Bins** | — | قائمة واحدة مؤقتة | منطقة عازلة قبل الفرز | — |

- **Fast Bins**: أحجام ثابتة (16, 24, 32, 40, 48, 56, 64, 72, 80, 88 بايت في x64)، نمط LIFO، ولا دمج للكتل المجاورة — تُدمج لاحقاً عبر `malloc_consolidate()` عند امتلائها أو عند طلب كتلة كبيرة.
- **Small Bins**: قوائم FIFO، يُدمج الجيران الحرة لتكوين كتل أكبر.
- **Large Bins**: مرتبة بالحجم، تستخدم خوارزمية Best-fit لتقليل التجزئة — بحثها أبطأ.

### دورة حياة الكتل المحررة

عند استدعاء `free(ptr)`:

1. يُتحقق من حجم الكتلة: إن كان **≤ 88** توضع في Fast Bins، وإلا توضع في Unsorted Bins ثم تُفرز لاحقاً (small/large).
2. عند التخصيص بـ `malloc()` يكون ترتيب البحث (منذ glibc 2.26):
   **tcache ← fastbins ← unsorted bins ← small/large bins**.

### مثال عملي

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    void *a = malloc(32);     // كتلة صغيرة → fast bin
    void *b = malloc(1024);   // كتلة كبيرة → unsorted bin
    void *c = malloc(2048);   // كتلة كبيرة أخرى

    free(a);   // تذهب إلى fast bin
    free(b);   // تذهب إلى unsorted bin
    free(c);   // تذهب إلى unsorted bin

    malloc(40); // إذا وجدت حجماً مناسباً تأخذ من fast bin
    return 0;
}
```

> يمكنك رؤية هذا بوضوح باستخدام **GDB مع pwndbg** (`heap bins` و `heap chunks`) — أداة لا غنى عنها لفهم سلوك الـ allocator.

## tcache — Thread-Local Cache

في **glibc 2.26** أُدخلت آلية **tcache** لتحسين أداء التخصيصات الصغيرة في البيئات متعددة الخيوط (Multi-Threaded).

**المشكلة التي حُلّت**: في البيئات متعددة الخيوط، كان الوصول للذاكرة المشتركة (main arena) يسبب **contention** (تنافساً) على القفل (mutex)، مما يبطئ الأداء. فخُصص **مخبأ محلي لكل thread** يدير كتلته المحررة بشكل مستقل دون قفل — مما قلل زمن التخصيص بنسبة 70-80%.

### خصائص tcache

- 64 قائمة لكل thread (بغض النظر عن المعمارية).
- أكبر حجم كتلة: **1032** بايت في x64.
- كل قائمة تحوي **7** كتل كحد أقصى.

**عند التخصيص**: إذا كان الحجم مناسباً (≤ 1032)، يبحث في قائمة tcache المناسبة — إن وجد كتلة يزيلها (LIFO) ويعيدها؛ وإلا يلجأ للـ Bins التقليدية.

**عند التحرير**: إذا كانت القائمة غير ممتلئة (< 7)، يُضيفها للـ tcache؛ وإلا تُرسل للـ Fast Bins أو Unsorted Bins.

## أشهر الثغرات من سوء التخصيص

### 1. Heap Overflow

نوع من تجاوز السعة (Buffer Overflow) في منطقة الـ Heap، يحدث عند كتابة بيانات تفوق الحجم المخصص. يسبب:

**أ. تلف البيانات المجاورة:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    char *buffer1 = (char*)malloc(10);
    char *buffer2 = (char*)malloc(10);

    strcpy(buffer2, "SECRET");

    strcpy(buffer1, "HEAP-OVERFLOW-ATTACK"); // تجاوز سعة buffer1

    printf("buffer2 = %s\n", buffer2); // تلفت بسبب التجاور
    free(buffer1);
    free(buffer2);
    return 0;
}
```

> **تنبيه**: النتيجة الفعلية تعتمد على تخطيط الـ chunks الحقيقي — في glibc الحديثة قد يتلف الـ metadata أولاً قبل user data، وقد يُكتشف الفساد عند `free`. لا تعتمد حرفياً على الناتج النظري.

**ب. تعديل الـ metadata:**

```c
char *chunk1 = (char*)malloc(16);
char *chunk2 = (char*)malloc(16);

memset(chunk1, 'A', 32); // تجاوز السعة: 16 < 32

free(chunk2); // النظام يكتشف تلف metadata وقد يتوقف
```

**ج. تعديل مؤشر الدالة (Function Pointer):**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef void (*func_ptr)();

void normal_func() { printf("normal\n"); }
void hacked_func() { printf("hackin\n"); }

int main() {
    func_ptr *fptr = (func_ptr*)malloc(sizeof(func_ptr));
    char *buffer = (char*)malloc(16);

    *fptr = normal_func;

    // تجاوز سعة buffer لتعديل fptr
    strcpy(buffer, "AAAAAAAAAAAAAAAA"  // 16 حرف
                   "\x40\x10\x40");    // عنوان hacked_func

    (*fptr)(); // يتم استدعاء hacked_func
    return 0;
}
```

> **تنبيه واقعي**: عنوان `hacked_func` هنا مثبت يدوياً للتبسيط — عملياً تحتاج الترجمة بـ `-no-pie` مع تعطيل ASLR حتى تثبت العناوين هكذا في الذاكرة.

### 2. Use After Free (UAF)

يحدث عندما يستخدم البرنامج مؤشراً إلى ذاكرة **بعد تحريرها** بـ `free()`:

**أ. تلف البيانات:**

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *ptr1 = (int*)malloc(sizeof(int));
    *ptr1 = 100;
    printf("the basic value: %d\n", *ptr1); // 100

    free(ptr1);

    // كتلة جديدة بنفس الحجم
    int *ptr2 = (int*)malloc(sizeof(int));
    *ptr2 = 200;

    // استخدام المؤشر المحرر (UAF)
    *ptr1 = 300; // كتابة على ذاكرة محررة

    printf("the new ptr2: %d\n", *ptr2); // قيمة غير متوقعة
    return 0;
}
```

**ب. تنفيذ كود خبيث (الجزء الأخطر):**

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    void (*printFunc)(); // مؤشر لدالة
} Object;

void legitFunc() { printf("legit function\n"); }
void evilFunc()  { printf("exploit function\n"); }

int main() {
    Object *obj = (Object*)malloc(sizeof(Object));
    obj->printFunc = legitFunc;
    obj->printFunc(); // استدعاء الدالة العادية

    free(obj); // تحرير المؤشر

    // تخصيص كتلة بنفس الحجم تحت سيطرة المهاجم
    unsigned long *fake = (unsigned long*)malloc(sizeof(Object));
    *fake = (unsigned long)evilFunc;

    obj->printFunc(); // UAF → يستدعي evilFunc
    return 0;
}
```

> **ربط RE**: هذا السيناريو هو بالضبط أساس **tcache poisoning** — أسهل هجمات الـ Heap وأكثرها شيوعاً. إعادة تخصيص كتلة محررة ووضع عنوان خبيث فيها هو ما تفعله هذه الهجمات.

### 3. Double Free

تحرير نفس المؤشر مرتين دون إعادة تخصيص بينهما:

**أ. تلف الـ Heap:**

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *ptr = (int*)malloc(sizeof(int));
    *ptr = 10;

    free(ptr); // التحرير الأول - صحيح

    // ... تعليمات ...

    free(ptr); // التحرير الثاني - خطأ: double free
    return 0;
}
```

**ب. تنفيذ كود خبيث:**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct Data {
    char buffer[32];
    void (*security_check)();
};

void normal_check()   { printf("normal_check\n"); }
void malicious_code() { printf("hackin code\n"); }

int main() {
    struct Data *d1 = (struct Data*)malloc(sizeof(struct Data));
    d1->security_check = normal_check;

    free(d1); // تحرير أولي

    // المهاجم يسيطر على الكتلة المحررة
    struct Data *d2 = (struct Data*)malloc(sizeof(struct Data));
    struct Data *d3 = (struct Data*)malloc(sizeof(struct Data));

    memset(d2->buffer, 'A', 32);
    d2->security_check = malicious_code;

    free(d1); // double free - تعديل الـ metadata

    struct Data *d4 = (struct Data*)malloc(sizeof(struct Data));
    d4->security_check(); // تنفيذ الكود الضار
    return 0;
}
```

> **ربط RE**: هذا المثال هو أساس **Fastbin Dup Attack** — إحداث دورة في قائمة fastbins عبر double free لتخصيص ذاكرة في موقع تتحكم به.

## لماذا هذا مهم؟

> تخيل أن **45% من الثغرات بين 2020-2023** كانت مرتبطة بمشاكل إدارة الذاكرة (حسب تقارير MITRE/Microsoft).

### الاحتياطات الوقائية

**1. إعادة تعيين المؤشرات بعد التحرير:**

```c
free(ptr);
ptr = NULL; // منع استخدام المؤشر بعد التحرير
```

**2. أدوات التحقق الديناميكي:**

```bash
valgrind --leak-check=full ./programName
```

**3. آليات حماية نظام التشغيل:**
- **ASLR**: عشوائية عناوين الذاكرة.
- **DEP/NX**: منع تنفيذ الكود من مناطق غير تنفيذية.

## ربط السلسلة بالهندسة العكسية

هذا المقال هو **الأساس الكامل لـ Heap Exploitation**:

1. **معرفة الـ Bins تحدد نوع الهجوم**: وجود كتل في tcache يعني هجماتها الأسهل، بينما fastbins تتطلب فهم الفروقات الدقيقة. عندما يحلل المحلل عينة، فهم حالة الـ heap يخبره بما هو ممكن.
2. **UAF → tcache poisoning**، **Double Free → fastbin dup**: كل ثغرة شرحتها هنا لها تقنية exploitation محددة تُبنى عليها.
3. **تحليل malware**: كثير من الـ malware يستخدم heap tricks لعزل الكود أو تجنب الحماية — فهم ما شرحته يسهل قراءة هذه التقنيات في الـ disassembly.

---

إن شاء الله تكون غطينا أغلب الأمور المتعلقة بالتخصيص الديناميكي. بعض الأمور لم نشرحها بعد (مثل الـ struct والـ typedef بالتفصيل) وستأتي في مقالات قادمة. إن أصبت فمن الله، وإن أخطأت فمن نفسي والشيطان.

