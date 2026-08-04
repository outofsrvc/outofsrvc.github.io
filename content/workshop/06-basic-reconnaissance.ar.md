---
title: "Basic Reconnaissance"
description: "مهارات الاستطلاع الأولي وجمع المعلومات عن الملفات."
date: 2026-03-21T12:00:00+03:00
slug: "basic-reconnaissance"
weight: 6
hex: "0x6"
stage: "deep-dive"
categories: [deep-dive, malware-analysis]
tags: [static-analysis, dynamic-analysis, tools, reconnaissance, network-analysis]
translationKey: "basic-recon"
ShowToc: true
TocOpen: false
draft: false
---

بسم الله الرحمن الرحيم.

دائماً عندما نريد تحليل أي ملف، نقوم بالخطوات التالية سواء كان الهدف من التحليل هو اكتشاف ما إذا كان الملف سليماً أم يحتوي على برمجيات خبيثة (Malware)، أو حتى إن كنا نريد كسر حمايته (Cracking):

## The 4 Stages of Analysis

1. التحليل الثابت الأساسي (Basic Static Analysis):
   هذا التحليل لا يتطلب خبرة تقنية عميقة، بل يعتمد بشكل كلي على أدوات تعمل بشكل تلقائي (مثل VirusTotal). يكشف لك هذا التحليل عن مؤشرات (Indicators) تبين ما إذا كان هذا الملف فايروساً أم لا. 
   > القاعدة: هذا التحليل يتم بدون تشغيل البرنامج (Without running the executable).

2. التحليل الديناميكي الأساسي (Basic Dynamic Analysis):
   مشابه لما سبقه، ولكن الفرق أنك هنا بحاجة إلى بيئة وهمية (Virtualization).
   > القاعدة: هذه الخطوة تتطلب تشغيل البرنامج (Running the executable) لترى آلية عمله وسلوكه الفعلي (Behavior).

3. التحليل الثابت المتقدم (Advanced Static Analysis):
   هذه المرحلة يتوجب أن يكون لديك خبرة كافية للتعامل مع برامج التفكيك (Disassemblers) كـ IDA Pro و Ghidra. نلخص هذه المرحلة بأنك سوف تحلل أكواد الأسمبلي (Assembly) وتفهم آلية عملها، أي الهيكلية التي يمشي بها البرنامج.

4. التحليل الديناميكي المتقدم (Advanced Dynamic Analysis):
   هذه المرحلة تتطلب خبرة تقنية لأنك سوف تتعامل مع المنقح (Debugger). لنشبهه بالـ Disassembler، ولكن الفرق بينهما هو أن الـ Debugger يسمح لك بأن تنفذ الكود وتمشي فيه سطراً بسطر في الذاكرة الحية، بينما الـ Disassembler لا ينفذ الكود (وهو أساس التحليل الثابت). من أفضل الـ Debuggers هو x64dbg.

---

## Basic Reconnaissance (الاستطلاع الأساسي)

في هذا المقال، سنناقش أول خطوتين من عملية التحليل، وما يعرف بالاستطلاع الأساسي: وهو جمع معلومات ومؤشرات أولية سريعة حول الملف قبل الغوص في التحليل العميق. وكما قلنا، هذه الخطوة لا تحتاج إلى خبرة تقنية معقدة، فهي عبارة عن استخدام أدوات بسيطة. سأذكر كل أداة، هدف استخدامها، وصورة لواجهتها.

### 1. Helpful Websites

* [VirusTotal](https://www.virustotal.com/): للتحقق من البصمة الرقمية للملف (Hashing) ومعرفة ما إذا كانت شركات مكافحة الفيروسات قد صنفته كخبيث.
  
![Virustotal](/assets/img/workshop/basic-recon/virustotal.png)
_شكل (1): واجهة موقع virustotal._


* [Hybrid-Analysis](https://www.hybrid-analysis.com/): خدمة توفر تشغيلاً تلقائياً للملف وتقدم تقريراً سريعاً عن سلوكه (سجل الملفات، العمليات، والشبكة).
  
![hybrid analysis](/assets/img/workshop/basic-recon/hybridanalysis.png)
_شكل (2): واجهة موقع hybrid-analysis._


* [CyberChef](https://gchq.github.io/CyberChef/): أداة لفك التشفير وتحليل البيانات (مثل Base64, XOR).
  
![Cyber Chef](/assets/img/workshop/basic-recon/cyberchef.png)
_شكل (3): واجهة موقع CyberChef._


---

### 2. Information Gathering: Static

* [Detect It Easy (DIE)](https://github.com/horsicq/Detect-It-Easy): لمعرفة نوع الملف ونوع المترجم (Compiler) وهل هو ملف مضغوط بـ Packer أم لا. من أهم المؤشرات في هذا البرنامج هو الـ Entropy.
  
![Detect It Easy](/assets/img/workshop/basic-recon/die.png)
_شكل (4): واجهة تطبيق DIE._


* [FLOSS](https://github.com/mandiant/flare-floss): أداة قادرة على استخراج النصوص التي يحاول المبرمج تشفيرها داخل الكود (Obfuscated Strings).

![Floss](/assets/img/workshop/basic-recon/floss.png)
_شكل (5): أمر تنفيذ برنامج FLOSS._

  
* [PE-Bear](https://github.com/hasherezade/pe-bear): يستخدم لتحليل ترويسات الملف (PE Headers) واستكشاف الموارد (Resources) ومعرفة المكتبات (DLLs) والدوال التي يستدعيها البرنامج مثل CreateProcessA.
  
![PE Bear](/assets/img/workshop/basic-recon/pe-bear.png)
_شكل (6): واجهة تطبيق PE-Bear._


---

### 3. Information Gathering: Dynamic

* [Process Monitor (ProcMon)](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon): يقوم بمراقبة نشاط الملفات وسجل النظام (Registry) وحركة الشبكة في الوقت الفعلي.
  
![ProcMon](/assets/img/workshop/basic-recon/procmon.png)
_شكل (7): واجهة تطبيق ProcMon._


* [Process Explorer (ProcExp)](https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer): يستخدم لمراقبة العمليات النشطة حالياً في النظام واكتشاف أي عمليات مشبوهة.
  
![ProcExp](/assets/img/workshop/basic-recon/procexp.png)
_شكل (8): واجهة تطبيق ProcExp._


---

### 4. Network Monitoring

* [FakeNet-NG](https://github.com/mandiant/fakenet-ng): يستخدم لمحاكاة خدمات الإنترنت محلياً، مما يسمح للمحلل برؤية طلبات الشبكة (مثل HTTP, GET) دون الحاجة لاتصال حقيقي بالإنترنت لتجنب الخطر.
  
![FakeNet-NG](/assets/img/workshop/basic-recon/fakenet.png)
_شكل (9): صورة للملف الذي يتم انشاءه بواسطة عمل run للتطبيق بلاحقة .pcap الذي يتم فتحه على wireshark._


* [Wireshark](https://www.wireshark.org/): أداة لالتقاط وتحليل حركة الشبكة (Network Sniffing) لمعرفة ما إذا كان الملف يحاول التواصل مع خوادم خارجية للتحكم والسيطرة.
  
![WireShark](/assets/img/workshop/basic-recon/wireshark.png)
_شكل (10): واجهة تطبيق WireShark._
