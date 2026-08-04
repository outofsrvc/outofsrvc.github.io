---
title: "Recognizing C Code Constructs in Assembly"
description: "كيف تُترجم الأكواد البرمجية إلى لغة الآلة: المتغيرات، الحلقات، الشروط، المصفوفات، الهياكل، والقوائم المترابطة."
date: 2026-03-19T10:00:00+03:00
slug: "c-constructs-in-assembly"
weight: 4
hex: "0x4"
stage: "deep-dive"
categories: [deep-dive, programming]
tags: [reverse-engineering, malware-analysis, c-lang, x86-assembly]
translationKey: "c-and-asm"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

دائماً في بداية رحلة أي شخص عند التعامل مع لغة Assembly، يشعر بصعوبة كونها لغة Low-Level على عكس اللغات الأخرى كـ C أو Python. سنحاول في هذا المقال تبسيط أكواد الأسمبلي لكي نتمكن من فهمها بشكل أكبر إن شاء الله.

فكرة المقال هي أننا سنكتب كوداً بسيطاً بلغة C ونقوم بعمل Compile له ليصبح x86 Assembly، ثم نشرح الخطوات التي تحدث لنتمكن من تحليل الـ Control Flow.

> قبل البدء، إن لم يكن لديك علم كافي بـ لغة C و Assembly أنصحك بمراجعة السلاسل التالية: 
> 🔗 [سلسلة لغة C على شبكة شل](https://sh3ll.cloud/xf2/threads/4059/)
> 🔗 [سلسلة لفة Asm على شبكة شل](https://sh3ll.cloud/xf2/threads/4305/)

---

## 1. Global vs Local Variables


### Global Variables
بداية نقوم بكتابة هذا الكود البسيط بلغة C:

```c
int x = 1;
int y = 2;

void main() {
    x = x + y;
    printf("Total = %d\n", x);
}
```


ونقوم بعمل compile بالأمر التالي:

```bash
gcc -S -masm=intel -m32 -O0 filename.c -o filename.s
```

وعندما نقوم بفتح الملف الذي لاحقته `.s` يظهر التالي:

**x86 Assembly:**

```assembly
00401003  mov eax, dword_40CF60
00401008  add eax, dword_40C000
0040100E  mov dword_40CF60, eax      ; [1] تخزين النتيجة
00401013  mov ecx, dword_40CF60
00401019  push ecx
0040101A  push offset aTotalD        ; "total = %d\n"
0040101F  call printf
```

الـ global variables في الأسمبلي تكون عبارة عن عناوين ذاكرة (Memory Addresses) مثل: `dword_40CF60`.

### Local Variables
بالنسبة للـ Local Variables، فتكون عبارة عن إزاحة (Offset) معين عن الـ `ebp` أو الـ `esp` أو أي ريجستر آخر (مثل: `dword ptr [ebp-4]`). عندما نقوم باستخدام disassembler مثل IDA Pro (والذي سنشرحه بالتفصيل في قادم المقالات بإذن الله)، ستظهر المتغيرات المحلية بوضوح.

```c
void main()
{
int x = 1;
int y = 2;
x = x+y;
printf("Total = %d\n", x);
}
```

**x86 Assembly:**

```assembly
00401006  mov dword ptr [ebp-4], 1      ; [1]
0040100D  mov dword ptr [ebp-8], 2      ; [2]
00401014  mov eax, [ebp-4]
00401017  add eax, [ebp-8]
0040101A  mov [ebp-4], eax
0040101D  mov ecx, [ebp-4]
00401020  push ecx
00401021  push offset aTotalD ; "Total = %d\n"
00401026  call printf
```


> عندما نقوم باستخدام disassembler كـ `ida` والي بقادم المقالات رح نشرح عنه باذن الله هيك بيظهر الـ `local variables`.

```assembly
00401006  mov [ebp+var_4], 1        ; [1]
0040100D  mov [ebp+var_8], 2        ; [2]
00401014  mov eax, [ebp+var_4]
00401017  add eax, [ebp+var_8]
0040101A  mov [ebp+var_4], eax
0040101D  mov ecx, [ebp+var_4]
00401020  push ecx
00401021  push offset aTotalD ; "Total = %d\n"
00401026  call printf
```

---

## 2. If Statement

```c
int x = 1;
int y = 2;

if(x == y) {
    printf("x equals y.\n");
} else {
    printf("x is not equal to y.\n");
}
```

**x86 Assembly:**

```assembly
00401006  mov [ebp+var_8], 1
0040100D  mov [ebp+var_4], 2
00401014  mov eax, [ebp+var_8]
00401017  cmp eax, [ebp+var_4]       ; [1] تعليمة المقارنة
0040101A  jnz short loc_40102B       ; [2] قفز مشروط إلى else
0040101C  push offset aXEqualsY_     ; "x equals y.\n"
00401021  call printf
00401026  add esp, 4
00401029  jmp short loc_401038       ; [3] قفز غير مشروط لتخطي else
0040102B loc_40102B:                 ; قسم else
0040102B  push offset aXIsNotEqualToY ; "x is not equal to y.\n"
00401030  call printf
```

أول شيء سنواجهه هو تعليمة `CMP`، تليها قفزة مشروطة (`jnz`). إذا تحقق شرط القفزة المشروطة، ينتقل التنفيذ إلى كتلة `else`. أما إذا لم يتحقق، فيستمر الكود في المسار الطبيعي (fall-through) وهو كتلة `if` الأساسية، ثم ينفذ قفزة غير مشروطة لتجاوز كتلة `else`.

---

## 3. For Loop

```c
int i;
for(i = 0; i < 100; i++) {
    printf("i equals %d\n", i);
}
```

**x86 Assembly:**

```assembly
00401004  mov [ebp+var_4], 0         ; [1] التهيئة (i=0)
0040100B  jmp short loc_401016       ; [2] قفز للمقارنة
0040100D loc_40100D:                 ; منطقة التحديث (Update)
0040100D  mov eax, [ebp+var_4]       ; [3] 
00401010  add eax, 1                 ; زيادة العداد
00401013  mov [ebp+var_4], eax       ; [4]
00401016 loc_401016:                 ; منطقة المقارنة (Condition)
00401016  cmp [ebp+var_4], 64h       ; [5] 64h = 100
0040101A  jge short loc_40102F       ; [6] قفز مشروط للخروج من الحلقة
0040101C  mov ecx, [ebp+var_4]       ; محتوى الحلقة
0040101F  push ecx
00401020  push offset aID            ; "i equals %d\n"
00401025  call printf
0040102A  add esp, 8
0040102D  jmp short loc_40100D       ; [7] العودة للتحديث
```

1. يتم التهيئة بمتغير محلي داخل الـ for.
2. نرى قفزاً غير مشروط يأخذنا للمقارنة `CMP`.
3. بعدها قفز مشروط للخروج. إذا لم يتم الخروج، تُنفذ التعليمات داخل الـ for.
4. في النهاية، قفز غير مشروط يأخذنا لمكان التحديث (`increment` أو `decrement`).
5. تتكرر العملية حتى يتحقق شرط الخروج.


![For Loop in Graph](/assets/img/workshop/c-and-asm/graph-for-loop.png)
_شكل (1): مقتطف من برنامج IDA Pro._


---

## 4. While Loop

```c
int status = 0;
int result = 0;

while(status == 0) {
    result = performAction();
    status = checkResult(result);
}
```

**x86 Assembly:**

```assembly
00401036  mov [ebp+var_4], 0
0040103D  mov [ebp+var_8], 0
00401044 loc_401044:                 ; بداية الحلقة
00401044  cmp [ebp+var_4], 0         ; المقارنة
00401048  jnz short loc_401063       ; [1] قفز مشروط للخروج
0040104A  call performAction
0040104F  mov [ebp+var_8], eax
00401052  mov eax, [ebp+var_8]
00401055  push eax
00401056  call checkResult
0040105B  add esp, 4
0040105E  mov [ebp+var_4], eax       ; تحديث الحالة
00401061  jmp short loc_401044       ; [2] العودة لبداية الحلقة
```

شبيهاً بالـ `for` ولكن أسهل بعض الشيء. البداية تكون بمقارنة `CMP` يليها قفز مشروط للخروج من الـ `while`. إذا لم ينفذ القفز، يُنفذ الكود داخل الحلقة، وفي نهايته قفز غير مشروط لإعادة الدورة.
وهكذا حتى يتحقق شرط الـ `CMP` ويقفز خارجها وينتهي الـ loop.

---

## 5. Switch Statement

ممكن أن يتم تجميع الـ Switch بطريقتين بناءً على المجمع (Compiler):

### الطريقة الأولى: If Style

تكون مشابهة تقريباً للـ `if` المتسلسلة. نرى عدد مقارنات مساوٍ لعدد الحالات، وبعد كل مقارنة قفز مشروط لتنفيذ الأمر. وفي النهاية قفز غير مشروط للـ `default`.

```c
switch(i) {
    case 1: printf("i = %d", i+1); break;
    case 2: printf("i = %d", i+2); break;
    case 3: printf("i = %d", i+3); break;
    default: break;
}
```

**x86 Assembly:**

```assembly
00401013  cmp [ebp+var_8], 1
00401017  jz short loc_401027        ; [1] Case 1
00401019  cmp [ebp+var_8], 2
0040101D  jz short loc_40103D        ; Case 2
0040101F  cmp [ebp+var_8], 3
00401023  jz short loc_401053        ; Case 3
00401025  jmp short loc_401067       ; [2] Default

00401027 loc_401027:                 ; تنفيذ Case 1
00401027  mov ecx, [ebp+var_4]       ; [3]
0040102A  add ecx, 1
; ... (بقية التعليمات) ...
```


![If Style in Graph](/assets/img/workshop/c-and-asm/graph-if-style.png)
_شكل (2): مقتطف من برنامج IDA Pro._



### الطريقة الثانية: Jump Table

```c
switch(i) {
    case 1: printf("i = %d", i+1); break;
    case 2: printf("i = %d", i+2); break;
    case 3: printf("i = %d", i+3); break;
    case 4: printf("i = %d", i+3); break;
    default: break;
}
```

**x86 Assembly:**

```assembly
00401016  sub ecx, 1                 ; طرح 1 لأن الكومبايلر يبدأ من 0
00401019  mov [ebp+var_8], ecx
0040101C  cmp [ebp+var_8], 3         ; مقارنة بالحد الأقصى
00401020  ja short loc_401082        ; إذا تجاوز الحد، اذهب للـ default
00401022  mov edx, [ebp+var_8]
00401025  jmp ds:off_401088[edx*4]   ; [1] القفز المباشر عبر الجدول

; --- العناوين المستهدفة ---
0040102C loc_40102C:
; ...
00401042 loc_401042:
; ...
00401082 loc_401082:                 ; النهاية (Default/Exit)
00401082  xor eax, eax
00401087  retn

; --- جدول القفز ---
00401088 off_401088:                 ; [2]
00401088  dd offset loc_40102C
0040108C  dd offset loc_401042
00401090  dd offset loc_401058
00401094  dd offset loc_40106E
```

طريقة جدول القفز تعتمد على طرح قيمة للحصول على `index` يبدأ من الصفر، ثم مقارنة مع عدد الحالات (cases)، وإذا كان ضمن النطاق يتم القفز مباشرة باستخدام الجدول بناءً على المعادلة `[edx*4]`.
واذا لم يكن ضمن النطاق ينفذ فورا الـ `default`.

![Jump Style in Graph](/assets/img/workshop/c-and-asm/graph-jmp-table.png)
_شكل (3): مقتطف من برنامج IDA Pro._

---

## 6. Arrays

```c
int b[5] = {123, 87, 487, 7, 978};

void main() {
    int i;
    int a[5];
    for(i = 0; i < 5; i++) {
        a[i] = i;
        b[i] = i;
    }
}
```

**x86 Assembly:**

```assembly
00401021  mov edx, [ebp+var_18]
00401024  mov [ebp+ecx*4+var_14], edx ; [1] Local Array
00401028  mov eax, [ebp+var_18]
0040102B  mov ecx, [ebp+var_18]
0040102E  mov dword_40A000[ecx*4], eax ; [2] Global Array
```

الـ Memory Address الخاص بالمصفوفة يعتمد على تعريفها (Global أم Local). ويرافقها دائماً Register يعمل كـ Index مضروب بحجم عناصرها. (مثلاً: مصفوفة Integers يكون الـ Index مضروباً بـ 4 `[ecx * 4]`).
مع الانتباه إلى أن فهارس المصفوفة (index) تبدأ من `0`، أي أن العنصر الأخير يقع عند الفهرس `n-1`.

---

## 7. Structs & Linked Lists

### Struct

```c
struct my_structure {                ; [1]
    int x[5];
    char y;
    double z;
};

struct my_structure *gms;            ; [2]

void main() {
    gms = (struct my_structure *) malloc(sizeof(struct my_structure));
    test(gms);
}
```

**x86 Assembly:**

```assembly
00401053  push 20h                   ; حجم الـ Struct (32 bytes)
00401055  call malloc
0040105A  add esp, 4
0040105D  mov dword_40EA30, eax      ; تخزين العنوان الأساسي
00401062  mov eax, dword_40EA30
00401067  push eax                   ; [1] تمرير المؤشر للدالة
00401068  call sub_401000
```

يتم تعريف الـ `struct` كـ متغير ويتم إضافة القيم الي جواه على حسب المتغيرات التي يحتويها وحسب استدعاء الدالة.
عند تعريف الـ `struct` وحجز مساحة له بـ `malloc`، يعطينا العنوان الأساسي (Base Address) الذي هو بداية الـ struct.

### Linked List

```c
struct node {
    int x;
    struct node * next;
};
typedef struct node pnode;

void main() {
    pnode * curr, * head;
    int i;
    head = NULL;
    
    for(i=1; i<=10; i++) {           ; [1] بناء الـ Nodes
        curr = (pnode *)malloc(sizeof(pnode));
        curr->x = i;
        curr->next = head;
        head = curr;
    }
    
    curr = head;
    while(curr) {                    ; [2] التنقل بين الـ Nodes
        printf("%d\n", curr->x);
        curr = curr->next;
    }
}
```

**x86 Assembly:**

```assembly
0040107E  mov [esp+18h+var_18], 8    ; حجم الـ Node
00401085  call malloc
0040108A  mov [ebp+var_4], eax       ; eax يحمل عنوان الـ Node الجديد
0040108D  mov edx, [ebp+var_4]
00401090  mov eax, [ebp+var_C]
00401093  mov [edx], eax             ; [1] curr->x = i
00401095  mov edx, [ebp+var_4]
00401098  mov eax, [ebp+var_8]
0040109B  mov [edx+4], eax           ; [2] curr->next = head
0040109E  mov eax, [ebp+var_4]
004010A1  mov [ebp+var_8], eax       ; head = curr
```

الفرق الجوهري بين الـ struct العادي والـ Linked List في الأسمبلي هو كثرة عمليات `mov` التي تدل على تكوين الروابط (Links) بين الـ Nodes كما يظهر في الأسطر المعلمة بـ `[1]` و `[2]`.
