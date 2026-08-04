---
title: "RE Toolkit: Disassemblers & Debuggers"
description: "إعداد بيئة العمل والأدوات: Ghidra، x64dbg، وIDA."
date: 2026-03-22T10:00:00+03:00
slug: "re-toolkit"
weight: 7
hex: "0x7"
stage: "deep-dive"
categories: [deep-dive, malware-analysis]
tags: [ida-pro, ghidra, x64dbg, disassembler, debugger]
translationKey: "re-tools"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

بعد الانتهاء من التحليل الأساسي (Basic Analysis)، ننتقل إلى التحليل العميق (Advanced Analysis). في هذا المقال سنناقش أهم برامج الـ Disassemblers والـ Debuggers المستخدمة لتحليل الملفات بشرح بسيط وسريع. وعند التطبيق العملي، سنتعرف على هذه الأدوات بشكل أفضل وأعمق.

---

## برامج التفكيك (Disassemblers)

تُستخدم هذه الأدوات لقراءة البرنامج دون تشغيله (Static Analysis)، وهي تشبه قراءة الخريطة لتحديد الاتجاهات وتضاريس المنطقة قبل بداية الرحلة.

* [IDA Free](https://hex-rays.com/ida-free/) (سنستخدمها في الورشة)
  
![IDA Free](/assets/img/workshop/re-tools/ida.gif)
_شكل (1): واجهة تطبيق IDA Freeware._


* [Ghidra](https://ghidra-sre.org/)
  

### فلسفة استعمال الأداتين سوياً:
* IDA (الرادار): واجهته هي الأفضل والأسرع في التنقل بين الدوال والمخططات البيانية (Graph View). هي الأداة التي نبدأ بها لنفهم "أين" نذهب وما هو الهيكل العام للبرنامج.
* Ghidra (التعمق): نسخة IDA المجانية لها قيود، مثل عدم دعم بعض المعماريات كـ ARM، وعدم وجود Decompiler (محول الكود إلى لغة C) لبعض الملفات. على عكس Ghidra الذي يوفر هذه الميزات القوية وبشكل مجاني تماماً (Open Source).

### Visual Modes (أنماط العرض)
* Graph Mode: يعرض تدفق التحكم (Control Flow) كمخطط بياني يسهل تتبع القفزات والقرارات البرمجية (If/Else, Loops).
* Text Mode: يعرض كود الأسمبلي بشكل متسلسل وتقليدي من الأعلى للأسفل.

### الوظائف المتقدمة (Advanced Functions)
تسمح للمحلل بالانتقال إلى مراجع الكود Xrefs (Cross-References) لمعرفة أين يتم استدعاء نصوص أو دوال معينة، وتتبع المسار العكسي وصولاً إلى نقطة البداية (Start Function أو Main).

### ⌨️ IDA Command CheatSheet

[IDA-Cheatsheet](https://malwareunicorn.org/workshops/idacheatsheet.html)

| Command (الاختصار) | Action (الإجراء)                                |
| :----------------- | :---------------------------------------------- |
| X                  | Jump to Xref (الانتقال إلى المراجع)             |
| G                  | Jump to address (الانتقال إلى عنوان ذاكرة محدد) |
| SHIFT + ; أو :     | Enter comment (إضافة تعليق على الكود)           |

---

## المنقحات (Debuggers)

* [x64dbg](https://x64dbg.com/) (سنستخدمها في الورشة)
  
![x64dbg](/assets/img/workshop/re-tools/x64dbg.png)
_شكل (2): واجهة تطبيق x64dbg._

تُستخدم هذه الأدوات لإجراء تحليل أعمق عبر تشغيل البرنامج فعلياً في بيئة محكومة لمراقبة سلوكه المخفي، والذي قد لا يظهر غالباً في الـ Disassembler بسبب تقنيات التشويش (Obfuscation).

### التحكم في التنفيذ (التلاعب بالمنطق)
تتيح هذه الأدوات للمحلل "التلاعب" بمسار البرنامج في الذاكرة الحية. مثل تغيير قيم المسجلات (Registers) أو الأعلام (Flags) لتجاوز شروط معينة (Branch Statements) والوصول إلى أكواد مخفية أو تخطي شاشات التحقق من التفعيل (Cracking).

### ⌨️ x64dbg Command CheatSheet

| Command (الاختصار) | Action (الإجراء)                                    |
| :----------------- | :-------------------------------------------------- |
| ;                  | Enter comment (إضافة تعليق)                         |
| F2                 | Toggle Breakpoint (وضع/إزالة نقطة توقف)             |
| F7                 | Step Into (الدخول داخل الدالة)                      |
| F8                 | Step Over (تخطي الدالة والانتقال للسطر التالي)      |
| F9                 | Run (تشغيل البرنامج حتى نقطة التوقف التالية)        |
| Space              | Edit Instruction (تعديل تعليمة الأسمبلي في الذاكرة) |

---

### Keyboard Layout for IDA Free & x64dbg

![Keyboard layout](/assets/img/workshop/re-tools/keyboard-layout.png)
_شكل (3): خريطة للمفاتيح التي سنحتاج التعامل معها دائماً أثناء التحليل._
