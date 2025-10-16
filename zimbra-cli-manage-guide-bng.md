# 📧 Zimbra Mail Server: Ultimate CLI Management Guide  
> **For Sysadmins, DevOps, and Zimbra Managers — Tested on Zimbra 8.x/9.x/10.x**

---

## 📋 সূচিপত্র (Table of Contents)

1. [📌 ভূমিকা ও ব্যবহারের নির্দেশিকা](#-ভূমিকা-ও-ব্যবহারের-নির্দেশিকা)  
2. [🌐 নেটওয়ার্ক ও সিস্টেম তথ্য](#-নেটওয়ার্ক-ও-সিস্টেম-তথ্য)  
3. [👤 ইউজার অ্যাকাউন্ট ম্যানেজমেন্ট](#-ইউজার-অ্যাকাউন্ট-ম্যানেজমেন্ট)  
4. [📬 মেইল কিউ ম্যানেজমেন্ট (Queue)](#-মেইল-কিউ-ম্যানেজমেন্ট-queue)  
5. [🗑️ স্প্যাম ও কম্প্রোমাইজড অ্যাকাউন্ট হ্যান্ডলিং](#️-স্প্যাম-ও-কম্প্রোমাইজড-অ্যাকাউন্ট-হ্যান্ডলিং)  
6. [📊 মেইলবক্স সাইজ ও স্টোরেজ ম্যানেজমেন্ট](#-মেইলবক্স-সাইজ-ও-স্টোরেজ-ম্যানেজমেন্ট)  
7. [🔐 অ্যাডমিন রাইটস, সিকিউরিটি ও ভিউ মেইল রেস্ট্রিকশন](#-অ্যাডমিন-রাইটস-সিকিউরিটি-ও-ভিউ-মেইল-রেস্ট্রিকশন)  
8. [⚙️ সার্ভিস ম্যানেজমেন্ট, রিস্টার্ট, ট্রাবলশুটিং](#️-সার্ভিস-ম্যানেজমেন্ট-রিস্টার্ট-ট্রাবলশুটিং)  
9. [📜 স্ক্রিপ্ট ও অটোমেশন (Bash)](#-স্ক্রিপ্ট-ও-অটোমেশন-bash)  
10. [📌 এমার্জেন্সি কমান্ড (Emergency Cheatsheet)](#-এমার্জেন্সি-কমান্ড-emergency-cheatsheet)  
11. [📚 রেফারেন্স ও আরও পড়ুন](#-রেফারেন্স-ও-আরও-পড়ুন)

---

## 📌 ভূমিকা ও ব্যবহারের নির্দেশিকা

Zimbra একটি শক্তিশালী ওপেন সোর্স ইমেইল সার্ভার। ওয়েব UI ভালো হলেও, **CLI (Command Line Interface)** হলো দ্রুত, শক্তিশালী এবং বাল্ক/অটোমেশনের জন্য অপরিহার্য।

> ⚠️ **সতর্কতা**:  
> - প্রায় সব কমান্ড `zimbra` ইউজার হিসেবে রান করুন।  
> - `postsuper -d ALL` — একবার ডিলিট হলে ফিরবে না!  
> - পাসওয়ার্ড পরিবর্তন বা অ্যাকাউন্ট ডিলিট করার আগে নিশ্চিত হোন।

```bash
su - zimbra   # Always start with this
```

---

## 🌐 নেটওয়ার্ক ও সিস্টেম তথ্য

### পাবলিক IP চেক করুন

```bash
curl ipinfo.io/ip
```

> স্প্যাম ব্ল্যাকলিস্ট চেক বা আউটগোয়িং মেইল সোর্স ভেরিফাই করতে কাজে লাগে।

### ডিস্ক স্পেস চেক

```bash
df -h
lsblk
```

> `/opt/zimbra` পার্টিশন ফুল হলে Zimbra স্টার্ট হবে না!

---

## 👤 ইউজার অ্যাকাউন্ট ম্যানেজমেন্ট

### ➕ নতুন ইউজার তৈরি

```bash
zmprov ca sumon@mydomain.com 'P@ssw0rd!2024'
```

> `ca` = Create Account — শক্তিশালী পাসওয়ার্ড ব্যবহার করুন।

---

### ➖ ইউজার ডিলিট

```bash
zmprov da sumon@mydomain.com
```

> ⚠️ মেইলবক্স, অ্যালিয়াস, সবকিছু মুছে যাবে — অপরিবর্তনীয়!

---

### 🔐 পাসওয়ার্ড পরিবর্তন

```bash
zmprov sp sumon@mydomain.com 'NewP@ss!2024'
```

> `sp` = Set Password — ইউজারকে জানান পরিবর্তনের পর।

---

### 🔒 অ্যাকাউন্ট লক/আনলক

```bash
# Lock (Disable Login)
zmprov ma username@domain.com zimbraAccountStatus locked

# Unlock (Enable Login)
zmprov ma username@domain.com zimbraAccountStatus active

# Other: maintenance, closed, pending
```

> কম্প্রোমাইজড অ্যাকাউন্টের জন্য `locked` ব্যবহার করুন।

---

### 📜 সব ইউজার লিস্ট

```bash
zmaccts
```

> অ্যাকাউন্ট নাম, স্ট্যাটাস (active/locked), ক্রিয়েশন ডেট দেখায়।

---

### 📜 সব অ্যাডমিন লিস্ট

```bash
zmprov -l gaaa
```

> `gaaa` = Get All Admin Accounts

---

### 👑 ইউজারকে অ্যাডমিন বানান

```bash
zmprov ma sumon@mydomain.com zimbraIsAdminAccount TRUE
```

---

### 👤 অ্যাডমিনকে সাধারণ ইউজারে নামান

```bash
zmprov ma sumon@mydomain.com zimbraIsAdminAccount FALSE
```

---

## 📬 মেইল কিউ ম্যানেজমেন্ট (Queue)

> ⚠️ এই কমান্ডগুলো `zimbra` ইউজার হিসেবে রান করুন — না হলে `postsuper: fatal: use of this command is reserved for the superuser` এরর আসবে।

### 🔍 মেইল কিউ দেখুন

```bash
mailq
# or
postqueue -p
# or full path
/opt/zimbra/common/sbin/postqueue -p
```

> Queue ID, Size, Time, Sender, Recipient দেখায়।

---

### 🔄 সব মেইল আবার ট্রাই করুন (Re-queue)

```bash
postqueue -f
```

> DNS/নেটওয়ার্ক ফিক্স করার পর ব্যবহার করুন।

---

### 🗑️ নির্দিষ্ট মেইল ডিলিট

```bash
postsuper -d 4A1B2C3D
```

> Queue ID ব্যবহার করুন — `mailq` থেকে পাবেন।

---

### 🗑️ সব মেইল ডিলিট (⚠️ ইমার্জেন্সি)

```bash
postsuper -d ALL
```

> সব পেন্ডিং/ডিফার্ড মেইল মুছে দেবে — শুধু জরুরি অবস্থায়!

---

### 🗑️ নির্দিষ্ট রিসিপিয়েন্টের মেইল ডিলিট (সেফ ভার্সন)

```bash
/opt/zimbra/common/sbin/mailq | \
awk '/shakil\.hossen@reverie-bd\.com/ {gsub(/^[*!]+/, "", $1); print $1}' | \
grep '^[0-9A-Fa-f]' | \
xargs -r /opt/zimbra/common/sbin/postsuper -d
```

> ✅ Queue ID ভ্যালিডেট করে, `*!` সরায়, একাধিক ডিলিট করে।

---

### 🗑️ শুধু `deferred` কিউ থেকে মেইল ডিলিট

```bash
/opt/zimbra/common/sbin/postsuper -d ALL deferred
```

> স্প্যামে ভরা deferred কিউ ক্লিন করতে আদর্শ।

---

## 🗑️ স্প্যাম ও কম্প্রোমাইজড অ্যাকাউন্ট হ্যান্ডলিং

### 🔍 কোন ইউজার স্প্যাম পাঠাচ্ছে? (লগ থেকে)

```bash
cat /var/log/zimbra.log | sed -n 's/.*sasl_username=//p' | sort | uniq -c | sort -n
```

> আউটপুট: `[count] user@domain.com` — সবচেয়ে বেশি কাউন্ট = সম্ভাব্য কম্প্রোমাইজড।

---

### 🔐 কম্প্রোমাইজড অ্যাকাউন্টের পাসওয়ার্ড পরিবর্তন

```bash
zmprov sp sathish@www.sathish.com 'NewStrongP@ss!2024'
```

> ⚠️ কিউ ক্লিন করার আগে পাসওয়ার্ড পরিবর্তন করুন — নতুন স্প্যাম রোধ করতে।

---

### 🗑️ কম্প্রোমাইজড ইউজারের সব কিউ মেইল ডিলিট

```bash
mailq | awk '/sathish@www\.sathish\.com/ {print $1}' | xargs -n1 postsuper -d
```

> `xargs -n1` — একবারে একটি করে ডিলিট — নিরাপদ।

```
/opt/zimbra/common/sbin/postqueue -p | tail -n +2 | awk 'BEGIN { RS = "" } / user@example\.com/ { print $1 }' | tr -d '*' | /opt/zimbra/common/sbin/postsuper -d -
```
> `user@example` - replace your actual address

---

## 📊 মেইলবক্স সাইজ ও স্টোরেজ ম্যানেজমেন্ট

### 📏 একজন ইউজারের মেইলবক্স সাইজ

```bash
zmprov gmi user@domain.com
```

> `gmi` = Get Mailbox Info — সাইজ, আইটেম কাউন্ট, শেষ লগইন।

---

### 📏 শুধু সাইজ (MB তে)

```bash
zmmailbox -z -m user@example.com gms | awk '{print $3/1024/1024 " MB"}'
```

> বাইট → MB — সহজে বোঝার জন্য।

---

### 📂 ফোল্ডার অনুযায়ী সাইজ (Inbox, Sent ইত্যাদি)

```bash
zmmailbox -z -m user@example.com gaf
```

> `gaf` = Get All Folders — প্রতিটি ফোল্ডারের সাইজ দেখায়।

---

### 📊 সব ইউজারের মেইলবক্স সাইজ রিপোর্ট (স্ক্রিপ্ট)

> ✅ ফাইল: `allmailboxsize.sh`

```bash
#!/bin/bash
echo "=== Zimbra Mailbox Size Report ==="
echo "Generated on: $(date)"
echo "=================================="

for account in $(zmprov -l gaa 2>/dev/null); do
    size=$(zmmailbox -z -m "$account" gms 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "Mailbox size of $account = $size"
    else
        echo "ERROR: Cannot access mailbox for $account"
    fi
done
```

> 🔧 এক্সিকিউট করুন:

```bash
chmod +x allmailboxsize.sh
./allmailboxsize.sh > mailbox_report_$(date +%Y%m%d).txt
```

> অডিট/ব্যাকআপের জন্য ডেটেড ফাইলে সেভ করুন।

---

## 🔐 অ্যাডমিন রাইটস, সিকিউরিটি ও ভিউ মেইল রেস্ট্রিকশন

> Zimbra তে ডিফল্ট অ্যাডমিন সব মেইল দেখতে পারে — কিন্তু আপনি চাইলে এটা বন্ধ করতে পারেন!

### 🚫 অ্যাডমিনকে ইউজার মেইল দেখতে বাধা দিন (Delegated Admin Group)

#### ধাপ ১: ডিস্ট্রিবিউশন লিস্ট তৈরি

```bash
zmprov cdl restricted-admins@mydomain.com zimbraIsAdminGroup TRUE
```

#### ধাপ ২: UI কম্পোনেন্ট সেট (অপশনাল — UI কাস্টমাইজ)

```bash
zmprov mdl restricted-admins@mydomain.com \
zimbraAdminConsoleUIComponents accountListView \
zimbraAdminConsoleUIComponents DLListView \
zimbraAdminConsoleUIComponents domainListView \
zimbraAdminConsoleUIComponents serverListView \
zimbraAdminConsoleUIComponents globalConfigView
```

#### ধাপ ৩: গ্লোবাল রাইটস গ্র্যান্ট (ভিউ মেইল ছাড়া!)

```bash
zmprov grr global grp restricted-admins@mydomain.com +domainAdminRights
zmprov grr global grp restricted-admins@mydomain.com +adminConsoleAccountRights
zmprov grr global grp restricted-admins@mydomain.com +adminConsoleServerRights
zmprov grr global grp restricted-admins@mydomain.com +adminConsoleDLRights
# ... add more as needed — but AVOID "viewMail" or "adminLoginAs"
```

#### ধাপ ৪: ডোমেইন লেভেলে রাইটস

```bash
zmprov grr domain mydomain.com grp restricted-admins@mydomain.com +domainAdminRights
zmprov grr domain mydomain.com grp restricted-admins@mydomain.com -adminLoginAs
```

#### ধাপ ৫: নতুন অ্যাডমিন ইউজার তৈরি + গ্রুপে যোগ

```bash
zmprov ca newadmin@mydomain.com 'Pass123!' zimbraIsDelegatedAdminAccount TRUE
zmprov adlm restricted-admins@mydomain.com newadmin@mydomain.com
```

#### ধাপ ৬: ক্যাশে ফ্লাশ

```bash
zmprov fc all
```

> 📘 রেফারেন্স: [Zimbra Wiki - Restrict Admin View Mail](https://wiki.zimbra.com/wiki/Restrict_Admin_%27View_Mail%27)

---

## ⚙️ সার্ভিস ম্যানেজমেন্ট, রিস্টার্ট, ট্রাবলশুটিং

### 📊 সার্ভিস স্ট্যাটাস চেক

```bash
zmcontrol status
```

> mailbox, mta, ldap, logger, stats — সব সার্ভিসের স্ট্যাটাস দেখায়।

---

### 🔄 সব সার্ভিস রিস্টার্ট

```bash
zmcontrol restart
```

> কনফিগ পরিবর্তন বা সার্ভিস হ্যাং হলে ব্যবহার করুন।

---

### 🛠️ পারমিশন ফিক্স (ক্রাশ/ম্যানুয়াল এডিটের পর)

```bash
/opt/zimbra/libexec/zmfixperms -e -v
```

> `-e` = extended, `-v` = verbose — সব ফাইল/ফোল্ডার পারমিশন ঠিক করে।

---

### 💾 ডিস্ক ফুল? Zimbra স্টার্ট হচ্ছে না?

```bash
su - zimbra
uptime
zmcontrol status
df -h              # Check disk space
lsblk              # Check partitions
zmcontrol stop     # Stop cleanly
# ➤ /opt/zimbra/log/, /tmp, /var/log থেকে পুরনো ফাইল ডিলিট করুন
zmcontrol start
zmcontrol status
```

> কমন ফিক্স: `logrotate`, `zmlogswatchctl restart`, বা পুরনো ব্যাকআপ ডিলিট।

---

## 📜 স্ক্রিপ্ট ও অটোমেশন (Bash)

### 📩 ইউজার ইনবক্স থেকে মেইল ডিলিট (CLI)

```bash
# নির্দিষ্ট তারিখের মেইল লিস্ট
zmmailbox -z -m user@domain.com s -t message -l 50 "in:inbox date:01/01/2024"

# নির্দিষ্ট msgid ডিলিট
zmmailbox -z -m user@domain.com deleteMessage 431
```

> ফিশিং/স্প্যাম মেইল বাল্ক ডিলিটের জন্য আদর্শ।

---

### 📜 অডিট লগ মনিটরিং

```bash
tail -f /opt/zimbra/log/audit.log | grep "user@domain.com"
```

> অ্যাডমিন অ্যাকশন, লগইন, ডিলিট — রিয়েল-টাইমে মনিটর।

---

## 📌 এমার্জেন্সি কমান্ড (Emergency Cheatsheet)

| সমস্যা | সমাধান |
|--------|----------|
| Zimbra হ্যাং/অপ্রতিক্রিয় | `zmcontrol restart` |
| ডিস্ক ফুল | `df -h` → `rm -rf /opt/zimbra/log/*.log.202*` → `zmcontrol start` |
| স্প্যাম ব্লাস্ট | `zmprov ma user@domain.com zimbraAccountStatus locked` → `zmprov sp ...` → Clear Queue |
| কিউ জ্যাম | `postsuper -d ALL deferred` → `postqueue -f` |
| UI স্লো/অ্যাক্সেস নেই | CLI ব্যবহার করুন — `mailq`, `postsuper`, `zmprov` |
| পারমিশন ইস্যু | `/opt/zimbra/libexec/zmfixperms -e -v` |

---

## 📚 রেফারেন্স ও আরও পড়ুন

- [Zimbra CLI Reference (zmprov)](https://wiki.zimbra.com/wiki/Zmprov)  
- [Restrict Admin View Mail](https://wiki.zimbra.com/wiki/Restrict_Admin_%27View_Mail%27)  
- [Managing Spam in Zimbra](https://sathisharthars.wordpress.com/2013/11/11/managing-spam-mails-in-zimbra-mail-server/)  
- [Get All Mailbox Sizes](https://wiki.zimbra.com/wiki/Get_all_user%27s_mailbox_size_from_CLI)  
- [Zimbra Forums](https://forums.zimbra.org/)  
- [Zimbra Bugzilla](https://bugzilla.zimbra.com/)

---

## ✅ বেস্ট প্র্যাকটিস সারসংক্ষেপ

| কাজ | সুপারিশ |
|------|----------|
| **পাসওয়ার্ড পরিবর্তন** | `zmprov sp` + ইউজারকে জানান |
| **কিউ ডিলিট** | আগে Dry Run → Queue ID ভ্যালিডেট → তারপর ডিলিট |
| **অ্যাডমিন অ্যাকাউন্ট** | Delegated Admin Group + Least Privilege |
| **স্প্যাম হ্যান্ডলিং** | লক → পাসওয়ার্ড চেঞ্জ → কিউ ক্লিন |
| **স্ক্রিপ্ট** | স্টেজিং-এ টেস্ট → এরর হ্যান্ডলিং যোগ করুন |
| **লগ** | নিয়মিত `/var/log/zimbra.log` ও `audit.log` মনিটর করুন |

---

## 🎯 শেষ কথা

> “CLI হলো Zimbra অ্যাডমিনের সবচেয়ে শক্তিশালী অস্ত্র — বিশেষ করে যখন UI কাজ করে না।”

---

> ✍️ **প্রস্তুতকারক**: Sumon Paul
> 📅 **শেষ আপডেট**: 21-09-2025  
> 🌐 **সাপোর্টেড**: Zimbra 8.8.x, 9.x, 10.x (Open Source & Network Edition)


---

✅ **আপনার এখন Zimbra CLI ম্যানেজমেন্টে পুরোপুরি প্রস্তুত — স্প্যাম থেকে শুরু করে ডেইলি অ্যাডমিনিস্ট্রেশন পর্যন্ত!**

---

> 🙏 **ধন্যবাদ — আপনার Zimbra সার্ভার এখন আরও সুরক্ষিত, কার্যকর ও সহজে ম্যানেজযোগ্য।**  
> কোনো প্রশ্ন বা কাস্টমাইজেশন প্রয়োজন হলে জানান — আমি সাহায্য করব!

--- 
