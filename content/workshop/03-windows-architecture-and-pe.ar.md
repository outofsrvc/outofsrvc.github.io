---
title: "Windows Architecture & PE Format"
description: "فهم معمارية نظام ويندوز وكيفية تعامله مع البرامج."
date: 2026-03-18T10:00:00+03:00
slug: "windows-architecture-and-pe"
weight: 3
hex: "0x3"
stage: "foundations"
categories: [foundations, os-internals]
tags: [windows, pe-format, memory, architecture, os-concepts]
translationKey: "win-arch-and-pe"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

هل تساءلت سابقاً كيف تعمل البرامج داخل الأنظمة؟ أو هل خطرت لك فكرة: ما الفرق بين المستخدم العادي والمطور من حيث فهمه للأنظمة التي يتعامل معها؟ 

بإذن الله في هذا المقال سنوضح تنسيق الملفات التنفيذية في ويندوز ونتكلم عن بعض الأمور الهامة التي لا يجب أن نجهلها في معمارية نظام ويندوز. 

> قبل البدء، إن لم يكن لديك علم كافي بأنظمة التشغيل أنصحك بمراجعة السلسلة التالية: 
> 🔗 [سلسلة أنظمة التشغيل على شبكة شل](https://sh3ll.cloud/xf2/threads/4230/)

---

## تنسيق الملفات (PE file format)
تنسيق ملفات PE (Portable Executable) هو التنسيق الأصلي لملفات win32 مثل (dll, exe). يعتبر فهم هذا التنسيق حجر أساس لتحليل ملفات أنظمة Windows. 

يشتق تنسيق PE بعض مواصفاته من Unix COFF (Common Object File Format)، وهو أحد تنسيقات ملفات الكائنات (object files) الذي خلفه لاحقاً تنسيق ELF (Executable and Linkable Format).

شو يعني Portable Executable؟
يعني أن التنسيق شامل ومنتشر على منصة win32 بحيث يتعرف عليه محمل الملفات (PE Loader) بأي منصة تعمل بنظام win32 بغض النظر عن نوع المعالج.

### ما هي هيكلية هذا التنسيق؟
هذا التنسيق يُعرف بشكل أساسي من PE Header: هو المكون الأساسي الذي يعرّف ملفات PE بشكل جوهري. يتكون من عدة طبقات متتالية تؤدي وظائف محددة لضمان تحميل الملف وتشغيله بشكل صحيح في بيئة windows. 

يتكون PE Header من عدة أجزاء تحدد هويته وسلوكه:

#### 1. رأس الدوس (Dos Header)
هو أول 64 بايت من الملف وبيحتوي على المعلومات اللازمة لمعرفة أن هذا الملف بتنسيق PE: 
* e_magic: الرقم السحري "magic number" الذي يشير إلى الحرفين MZ أو 4d 5a بالهيكس.. نسبة للمهندس "Mark Zbikowski" وهما اللذان يحددان أن الملف بتنسيق PE.

![DOS Header Magic Number](/assets/img/workshop/windows/dos-magic.png)
_شكل (1): الرقم السحري._

* e_lfanew: يقع عند الإزاحة 0x3c جوا الـ DOS Header وهو عبارة عن عنوان يشير إلى مكان بدأ PE Header (الـ NT Header الجديد).

![e_lfanew Offset](/assets/img/workshop/windows/e-lfanew.png)

#### 2. بقايا الدوس (DOS Stub)
عبارة عن بقايا للـ DOS Header يأتي بعد الـ 64 بايت الأولى لرأس الـ DOS وهو منطقة ذاكرة تكون غالباً مليئة بالأصفار أو تحتوي على رسالة خطأ بسيطة تشير إلى أن هذا البرنامج لا يمكن تشغيله في وضع DOS (إلي كانت معماريتها 16-bit).

![DOS Stub Message](/assets/img/workshop/windows/dos-stub.png)
_شكل (2): صورة توضح شكل الـ DOS Stub._

#### 3. رأس النظام (NT Header / PE Header)
يتم تعريفه برمجياً كـ بنية IMAGE_NT_HEADER.

![IMAGE_NT_HEADER Structure](/assets/img/workshop/windows/img-nt-header.png)
_شكل (3): صورة توضح برمجة البنية._

ويتكون من ثلاثة أجزاء:
1. التوقيع (signature): يتكون من البايتات السحرية PE\0\0 أو بالهيكس 50 45 00 00 للتعريف بتنسيق الملف.

![IMAGE_NT_HEADER Signature](/assets/img/workshop/windows/nt-header-sign.png)
_شكل (4): صورة توضح التوقيع._


2. رأس الملف (File Header أو COFF Header): يصف الخصائص الأساسية للملف مثل: 
   * Machine: نوع المعالج.
   * NumberOfSections: عدد الأقسام الموجودة في الملف.
   * Characteristics: سمات الملف (مثل هل هو Exe أو DLL).

![File Header](/assets/img/workshop/windows/file-header.png)
_شكل (5): صورة توضح رأس الملف._

3. الرأس الاختياري (Optional Header): على الرغم من تسميته إلا أنه إلزامي لملفات PE ويحتوي على متغيرات بالغة الأهمية: 
   * AddressOfEntryPoint: نقطة دخول البرنامج.
   * ImageBase: العنوان المفضل لتحميل الملف في الذاكرة.
   * DataDirectory: قائمة مكونة من 16 عنصر بتشير إلى جداول هامة مثل جدول التصدير "export table" والاستيراد "import table".

![Optional Header](/assets/img/workshop/windows/optional-header.png)
_شكل (6): صورة توضح الرأس الاختياري._

#### 4. جدول الأقسام (Section Table / Headers)
مثل الخريطة توضح كيفية تقسيم البيانات وتوزيعها في الذاكرة، يأتي هذا الجدول مباشرة بعد الـ Headers وقبل البيانات الفعلية للأقسام. 
يتكون جدول الأقسام من مصفوفة تسمى IMAGE_SECTION_HEADER حيث أن كل عنصر في هذه المصفوفة يصف قسم معين بالملف. وتبلغ مساحة كل عنصر في هذا الجدول 0x28 بايت. 

ويحتوي كل إدخال بالجدول على تفاصيل مهمة: 
* Name: اسم القسم (مثل .text او .data).
* VirtualSize: الحجم الفعلي للقسم عند تحميله بالذاكرة.
* VirtualAddress: العنوان الذي سيتم وضع القسم فيه في الذاكرة الافتراضية. 
* SizeOfRawData: حجم القرص على الهارد ديسك.
* Characteristics: سمات القسم (مثل قابل للقراءة أو الكتابة أو التنفيذ).

#### 5. الأقسام (Sections)
بتجي بعد الـ section table. ما هي هذه الأقسام؟ 
* .text: بيحتوي على كود البرنامج القابل للتنفيذ.
* .rdata: بيحتوي على البيانات القابلة للقراءة فقط (مثل Strings و Constants). 
أحياناً يتم تقسيم الـ rdata إلى قسمين وغالباً نرى ذلك في ملفات الـ DLL: 
  * .idata: ويكون للـ imports directory.
  * .edata: ويكون للـ exports directory وهذا قسم مهم في ملفات dll لربط Functions المصدر بأسماء أو أرقام تعريفية.
* .data: بيحتوي على البيانات والمتغيرات التي تم تهيئتها.
* .rsrc: بيحتوي على الموارد كالصور والأيقونات الخاصة بالبرنامج. 

![Sections View](/assets/img/workshop/windows/sections-table.png)
_شكل (7): صورة توضح الأقسام._

---

## ما هو الفرق بين ملفات Exe vs DLL؟ 
* ملفات Exe: تتطلب وجود دالة تسمى Main يقوم الـ OS Loader باستدعاء هذه الدالة عندما تكون العملية الجديدة جاهزة. وهذه الملفات تعمل بشكل مستقل؛ حيث يقوم النظام بإنشاء عملية جديدة ومساحة افتراضية مخصصة لها. 
* ملفات DLL: غالباً تحتوي على دالة تسمى DllMain يتم تنفيذ الكود بشكل مباشر بمجرد تحميل مكتبة dll المناسبة في الذاكرة. ولا يمكنها العمل بشكل مستقل بل يجب تحميلها داخل مساحة افتراضية لعنوان موجود مسبقاً. لماذا؟ لأن العملية قد تحتاج إلى functions توفرها ه‍ذه المكتبة.

أي باختصار: بينما يمثل ملف exe البرنامج الذي يبدأ العملية، يمثل ملف الـ dll المكتبة التي تمد هذه العملية بالدوال. 

---

## معمارية نظام ويندوز (Windows Internals)
ونعود لنتساءل ما الذي يميز المطور في الأنظمة عن المستخدم؟
ببساطة معرفة أمور النظام الداخلية التي لا تهم المستخدم العادي التي يطلق عليها Windows Internals.

الـ windows internals هي عبارة عن الـ Concepts الي لازم تكون بتعرف تتعامل معها بنظام ويندوز، أي حدا رح يتعلم موضوع low-level أو يبرمج شي low-level لازم يكون عنده خبرة بهي النقاط:

### 1. العمليات (Processes)
بتعني أي برنامج هو قيد التنفيذ.
العمليات الها ستراكشر بالـكيرنال اسمو KProcess (والتي يُطلق عليها بالمصطلح العام Process Control Block - PCB) الـكيرنال بيستخدموا مشان يتحكم بالـ operations تبع هي البروسيس.

بالـميموري الوضع مختلف، بيكون في شي اسمه EProcess structure وهو غير الـ KProcess; الـ EProcess بتحتوي ع كتير معلومات أهمها:
- Process id: رقم العملية.
- exe name: اسم ملف التنفيذ المرتبطة فيه هي العملية.
- dll files: ملفات الـ dll المرتبطين بهي العملية.
- PCB: ستراكشر الـكيرنال المرتبط بهي العملية.

![EProcess](/assets/img/workshop/windows/eprocess.png)
_شكل (8): صورة توضح ارتباط العمليات ببعضهم البعض._

هدول الـ EProcesses موجودين بالـ kernel memory على شكل double-linked يعني كل بروسيس الها forward link و back link. وطبعاً في كتير برامج بتوصل لهي الـ EProcess متل Process Explorer.

<div dir="ltr" style="text-align: center;">

![Process Explorer View](/assets/img/workshop/windows/process-explorer.png)
_شكل (9): صورة توضح عرض من داخل البرنامج._
</div>

### 2. الخيوط (Threads)
باختصار هي الشي الي ويندوز بيقوم بتنفيذو. أما بالنسبة للـ Process: هي بتحتوي بداخلها threads.
البروسيس ما هي الي بتشغل الكود بل لازم تحتوي على الأقل على one thread ليشغل الكود على المعالج، بس ما بيمشي الحال انو الثريد تشتغل لحالها إذا ما كانت موجودة داخل بروسيس.

الثريد الها 3 حالات ممكن تكون فيها:
- Running: يعني شغالة أو قيد التنفيذ.
- Ready: جاهزة منتظرة البروسيسور ليشغلها.
- Blocked/Suspended: توقفت أو انحظرت خلينا نقول جراء مقاطعة (interrupt).

![Threads Memory Status](/assets/img/workshop/windows/threads-status.png)
_شكل (10): صورة توضح حالات الخيوط._

بما أن البروسيس ممكن تحتوي على أكتر من thread. هل بيكونوا معزولين عن بعض؟
لا هني بيعملوا shared للميموري يعني الـ threads بيتشاركوا الـ resources والـ address spaces (متل .data section و .code section). بس بيضل لكل thread الها stack و registers خاصين فيها.

<div dir="ltr" style="text-align: center;">

![Threads Memory Sharing](/assets/img/workshop/windows/thread-memory.png)
_شكل (11): صورة توضح مشاركة الذاكرة بين الخيوط._
</div>

### 3. الذاكرة الافتراضية (virtual memory / virtual address space)
أي exe.file رح نشغلو رح يكون اله بروسيس وهي البروسيس الها address space خاص فيها.
الـ address space بياخد range من 00000000 إلى ffffffff وبيقسم هاد الرنج لقسمين كل قسم 2Gb: قسم للـ user mode وقسم للـ kernel mode.

![Physical vs Virtual](/assets/img/workshop/windows/virtual-vs-physical.png)
_شكل (12): صورة توضح المقارنة بين الجزء المادي والافتراضي من الذاكرة._

- الـ user mode الرانج تبعو: من 00000000 إلى 7fffffff
- والـ kernel mode: من 80000000 إلى ffffffff
*(مع العلم أن هاد الكلام بنظام الـ 32bit).*

### 4. العناوين الافتراضية والفيزيائية (virtual address VS physical address)
كل برنامج اله بروسيس وكل بروسيس الها ذاكرة افتراضية. ومتل ما منعرف انو كل exe file اله section بالذاكرة.
فـ لو عنا تطبيقين حطو .text و .data بعناوين عشوائية وشاء القدر انو تتشابه عناوين الـ .data، هل معنى هاد الشي انو رح يكون قسم الـ .data متشارك؟

لا، لأن الـ addresses الأساسية للبرامج موجودة بالرام الي هي الـ physical memory، والـ addresses الي عم نتعامل معها حالياً هي افتراضية virtually.
فـ لما البروسيس يصير بدا تقرا داتا بصير شي اسمو virtual-to-physical نظام التشغيل بيعمل handle للموضوع وبيوصل كل section بـ address معين بالـ RAM.

لازم نعرف:
الـ virtual address ما بتمثل موقع فعلي بالـرام؛ فـ بدل هالشي النظام بيحتفظ بـ pages لكل process (حتى يتم ترجمة العناوين الـ virtual لعناوين physical). وبنفس الوقت هاد الحكي مطبق على الثريدز وهي العملية اسمها virtual-to-physical translation.

بـ ويندوز عملوا حركة مشان ما كل بروسيس تحمل مقاطع خاصة الها من الرام. فعملوا ملفات بلواحق dll لهيك أي عملية لازم تعمل load لـ ntdll.dll و kernel32.dll وهدول عبارة عن مكاتب أو بالأصح (dynamic link library) يعني مكاتب ربط بيتم فيهن ربط أكتر من بروسيس الهن نفس الـ address. وأول بروسيس بتعمل mapping أو تقسيم لهي الـ dll بعناوين مشتركة بين كل الـ processes.

### 5. التزامن (Synchronization)
باختصار بيمنع انو يحصل simultaneous access (يعني كذا برنامج أو ثريد يستعملوا نفس الـملف أو نفس الـميموري بنفس الوقت).
فـ عملوا شي اسمه mutex أو كمان بينقال له lock: هو عبارة عن object منستعملوا بالبرمجة عموماً بيمنع حصول simultaneous access.

مثال يوضح الأمور شوي:
لو عنا عمليتين مختلفتين وعلى فرض بدهم يعملوا write بالميموري بنفس الوقت شو بصير ؟
تخيل الميموري مسؤول عنها police officer، هاد الضابط ما بيخلي حدا يفوت يعمل write إلا ليكون معه الـ mutex. يعني بتجي عملية معها الـ mutex object بتكتب بالميموري وبتترك الـ mutex، بتجي العملية التانية بتاخد هاد الـ mutex وبتكتب بالميموري.

### 6. الخدمات (Services)
الـسيرفس بتسمحلنا نشغل long running executable apps.
رح تضل شغالة بالـ background بتشتغل بـ session خاصة بالويندوز اسم الـ session هو svchost.exe (وهو عبارة عن host للـ services، هي الـسيرفايسس بتكون مجدولة وبينعملها run من windows service manager بدون تدخل اليوزر فيها).

### 7. الريجستري (Registry)
عبارة عن database بيتخزن فيها إعدادات نظام ويندوز والبرامج المتواجدة فيه.
الـ registry بتحتوي على عنصرين أساسيين:
1. Keys: هو عبارة عن الـ folders.
2. Values: عبارة عن الـ files.

بالـ registry في شي اسمه root keys أو HKEYs وهدول أهم المسارات الي منتعامل معهم:
- HKEY_CURRENT_USER
- HKEY_LOCAL_MACHINE
- HKEY_USERS
- HKEY_CLASSES_ROOT

طيب شو الشغلات المهمة الموجودة بالريجستري ؟
- قائمة البرامج والـسيرفايسس المنزلة على الـسيستم.
- الإعدادات تبع البرامج والـسيرفايسس.
- البرامج الي بتشتغل auto run after boot.
- الـ file associations (يعني اليوزر إذا بدو يفتح صفحة .html بيفتحها على chrome أو firefox مثلاً).
- منشوف الـ history تبع الـ usb devices والـ network adapter settings.
الملف الي بيكون فيه الـ functions تبع الريجستري اسمه advapi32.dll وكل الدوال الي الها علاقة بالريجستري بتبدأ بـ Reg.

### 8. ثوابت البرمجة في ويندوز (windows coding conventions)
بعض الثوابت بالـ Windows API، هي الثوابت بتسهل عليك قراءة الـ docs تبع مايكروسوفت أو توقع مثلاً شو بتعمل بعض function من اسمها.
windows data types:
- Byte -> 1Byte
- Word -> 2Bytes
- DWord -> 4Bytes
- QWord -> 8Bytes

مايكروسوفت بتعتمد على الـ prefix naming بالتسمية "prefix يعني القبلية" بتعرفنا انو مثلاً شو نوع الداتا تايب تبع متغير أو هيكلية معينة. وكل function فينا نستعلم عنها بـ MSDN - MicroSoft Developer Network وبآخر كل دالة منلاقي ملف الـ dll المتواجدة فيه هي الدالة.

### 9. المقابض (Handles)
الـ هاندل متل الـ بوينتر تقريباً...
الفرق انو البوينتر ممكن نعمل عليه عمليات رياضية بس الـ هاندل ما منقدر (يعني بس مناخد هاد المقبض ومنخزنه ومنستعمله متل ما هو بوقت لاحق).

على سبيل المثال:
بدنا نعمل call لـ func اسمها CreateWindowEx، هي الـدالة بس انعملها Call بتعمل Create لـ window وبترجع handle لهي الويندو. ولو بدي أعمل أي عملية على هي الويندو متل مثلاً ضيف أو أتحكم بـ زر أو أي شي لازم أعمل reference لهي الويندو عن طريق الـ handle.

### 10. دوال الشبكات (Network Functions in the API)
مايكروسوفت بتوفر 2 API للـ Networking:
1. low-level API:
هاد بيتعامل مع الـ sockets اسمه winsock. الـ socket عبارة عن handle بالـ endpoint للـ network communications.
*مثال:* في لعبة الأكواب الي بيتواصلوا فيها:

<div dir="ltr" style="text-align: center;">

![Sockets Analogy](/assets/img/workshop/windows/socket-analogy.png)
_شكل (13): صورة توضح مبدأ المرسل والمستقبل._
</div>

كل كوب هو عبارة عن سوكيت؛ كوب للسماع وكوب للكلام. فالسوكيت منعاملها كـ file: منقدر نعمل فيه write or read.

2. high-level API:
اسمه wininet بيسمح للمبرمجين انهم يستعملوا بروتوكولات الـ high-level متل http, ftp.
طبعاً هي المكتبة بتكون مواكبة للـ standards الخاصة بالبروتوكولات.. متل http، وفي كتير ملفات dll خاصة بال high-level متل: winhttp.dll, dnsapi.dll, urlmon.dll.

### 11. واجهة برمجة التطبيقات الأصلية (the native api)
لما منعمل Call لـ function من الـ win32api الـ function هي ما بتعمل الشي الي نحنا طلبناه مباشرة. بل بتحتاج تتواصل مع الـكيرنال مشان تقدر توصل للهاردوير.
فالـ userapps بتستعمل win32api من ملفات متل kernel32.dll وهي الملفات بتعمل Call لملف اسمو ntdll.dll: هو المسؤول عن الـ interactions بين الـ user mode والـ kernel mode.

الـ native api بتخلي الـ apps يتواصلوا مباشرة مع الـ ntdll.dll.

---

## المراجع
* Practical Malware Analysis 
* Windows Internals
* Malware Development for Ethical Hackers
