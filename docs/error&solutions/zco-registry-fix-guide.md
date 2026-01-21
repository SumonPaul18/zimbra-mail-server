# 📌 ZCO (Zimbra Connector for Outlook) – রেজিস্ট্রি কী যোগ করার সহজ গাইড  
**(Non-Technical Users-এর জন্য)**

## 🔍 সমস্যা কী?
Zimbra ইমেইল সার্ভারের সাথে Microsoft Outlook-এ কানেক্ট করার জন্য ZCO (Zimbra Connector for Outlook) নামে একটি টুল ব্যবহার করা হয়।  
কিছু কম্পিউটারে ZCO ঠিকমতো কাজ করছে না — এর কারণ হতে পারে **Windows রেজিস্ট্রি-তে কয়েকটি গুরুত্বপূর্ণ সেটিংস অনুপস্থিত**।

> 💡 **রেজিস্ট্রি কী?**  
> এটি Windows অপারেটিং সিস্টেমের একটি গোপন ডাটাবেজ যেখানে অ্যাপ্লিকেশন ও সিস্টেম সেটিংস সংরক্ষিত থাকে। ZCO-কে ঠিকমতো কাজ করার জন্য এখানে কয়েকটি ছোট এন্ট্রি যোগ করা দরকার।

---

## ✅ আপনার করণীয়: রেজিস্ট্রি কী যোগ করুন

> ⚠️ **গুরুত্বপূর্ণ নোট:**  
> আপনার অ্যাকাউন্টে **Administrator (অ্যাডমিন) অ্যাক্সেস** থাকা আবশ্যিক।  
> যদি আপনি নিজের কম্পিউটার না হন (যেমন: কোম্পানির কম্পিউটার), তাহলে **আপনার IT ডিপার্টমেন্টকে এই গাইডটি শেয়ার করুন**।

---

## 📋 ধাপ ১: আপনার Outlook টাইপ চেক করুন

প্রথমে জেনে নিন আপনার Outlook কোন ধরনের:

### ক) **MSI Outlook** – সাধারণত কোম্পানি/অর্গানাইজেশনে ইনস্টল করা হয়।  
### খ) **Click-to-Run Outlook** – যদি আপনি Microsoft 365 (পূর্বে Office 365) ব্যবহার করেন।

> 🔎 **কীভাবে চেক করবেন?**  
> Outlook খুলুন → **File** → **Office Account** → **About Outlook**  
> যদি লেখা থাকে: **"Click-to-Run"**, তাহলে আপনার Click-to-Run Outlook।  
> না হলে, সম্ভবত MSI Outlook।

---

## 📋 ধাপ ২: আপনার Windows ও Outlook আর্কিটেকচার চেক করুন

- **64-bit Windows?** → প্রায় সব আধুনিক কম্পিউটারে 62-bit Windows চলে।  
- **32-bit Outlook?** → বেশিরভাগ ক্ষেত্রে আপনার Outlook 32-bit হয়ে থাকে, এমনকি 64-bit Windows-এও।

> নিশ্চিত হতে চাইলে Windows-এ **Settings > System > About** এ গিয়ে **"System type"** দেখুন।

---

## 📋 ধাপ ৩: সঠিক রেজিস্ট্রি পথ খুঁজুন

আপনার কনফিগারেশন অনুযায়ী নিচের যেকোনো একটি পথ প্রযোজ্য:

### 🔹 যদি **MSI Outlook** ব্যবহার করেন:

| আপনার সিস্টেম | রেজিস্ট্রি পথ |
|----------------|----------------|
| 64-bit Outlook + 64-bit Windows **অথবা** 32-bit Outlook + 32-bit Windows | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Windows Search\Preferences` |
| 32-bit Outlook + 64-bit Windows | `HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Windows\Windows Search\Preferences` |

### 🔹 যদি **Click-to-Run Outlook** ব্যবহার করেন:

| আপনার সিস্টেম | রেজিস্ট্রি পথ |
|----------------|----------------|
| 64-bit Outlook + 64-bit Windows **অথবা** 32-bit Outlook + 32-bit Windows | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\REGISTRY\MACHINE\Software\Microsoft\Windows\Windows Search\Preferences` |
| 32-bit Outlook + 64-bit Windows | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\REGISTRY\MACHINE\Software\Wow6432Node\Microsoft\Windows\Windows Search\Preferences` |

> 💡 **সহজ টিপস:**  
> যদি আপনি 64-bit Windows ব্যবহার করেন (যা ৯৫% ক্ষেত্রে সত্য), এবং Outlook 32-bit হয় (সাধারণ), তাহলে **দ্বিতীয় রো ব্যবহার করুন** (WOW6432Node বা Wow6432Node সহ)।

---

## 📋 ধাপ ৪: রেজিস্ট্রি এডিটর খুলুন এবং কী যোগ করুন

1. **Windows + R** চাপুন → বক্সে লিখুন `regedit` → **Enter** চাপুন।  
2. **"Yes"** ক্লিক করুন (যদি UAC পপআপ আসে)।  
3. উপরের ধাপ ৩-এ পাওয়া রেজিস্ট্রি পথটি একে একে খুলুন।  
   - উদাহরণ: `HKEY_LOCAL_MACHINE` → `SOFTWARE` → `WOW6432Node` → ...  
4. **Preferences** ফোল্ডারে রাইট-ক্লিক করুন → **New → DWORD (32-bit) Value**  
5. নিচের দুটি কী আলাদা আলাদাভাবে তৈরি করুন:

   | নাম (Name) | ভ্যালু (Value) |
   |------------|----------------|
   | `{D00FDE68-3E80-4f8c-899D-D9DD16BA7D1D}` | `1` |
   | `{FA9628A0-F223-4d5d-B314-E01BC8100572}` | `1` |

   > ⚠️ **নোট:**  
   > - কী নামগুলো **কপি-পেস্ট করুন** — স্পেলিং একদম ঠিক হতে হবে।  
   > - ভ্যালু হিসেবে `1` লিখুন (হেক্সাডেসিমেল নয়, ডিসিমেল)।

6. সবকিছু সেভ হয়ে গেলে **Regedit বন্ধ করুন**।  
7. **কম্পিউটার রিস্টার্ট করুন** (ঐচ্ছিক, কিন্তু সুপারিশকৃত)।

---

## 📋 ধাপ ৫: ফলাফল যাচাই করুন

- Outlook আবার খুলুন এবং ZCO ব্যবহার করে দেখুন।  
- যদি সমস্যা থাকে, তাহলে **IT সাপোর্টকে জানান**: “রেজিস্ট্রি কী ম্যানুয়ালি যোগ করা হয়েছে, কিন্তু সমস্যা অব্যাহত আছে।”

---

## 🛡️ নিরাপত্তা ও সতর্কতা

- **রেজিস্ট্রি এডিট করার আগে ব্যাকআপ নিন** (File → Export in Regedit)।  
- কোনো অপ্রয়োজনীয় কিছু ডিলিট/পরিবর্তন করবেন না।  
- অনিশ্চিত হলে IT টিমকে জড়িত করুন।

---

> ✅ **সারসংক্ষেপ:**  
> ZCO-র সমস্যা সমাধানের জন্য Windows রেজিস্ট্রিতে 2টি কী (`{D00F...}` ও `{FA96...}`) `1` ভ্যালু সহ যোগ করুন — আপনার Outlook ও Windows টাইপ অনুযায়ী সঠিক পথ ব্যবহার করে।

আশা করি এই গাইডটি আপনার সমস্যা সমাধানে সাহায্য করবে! 🙏  

---
