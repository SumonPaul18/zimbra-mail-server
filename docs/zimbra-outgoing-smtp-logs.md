### USER
 less /var/log/maillog |grep 

১। আমি উপরের মত কমান্ড ব্যবহার করে outgoing smtp server এর লগ দেখতে চাই।
২। sender & receiver দিয়ে নির্দিষ্ট মেইল লগ দেখতে চাই।


### ASSISTANT
আপনি যে কমান্ডটি লিখেছেন (`less /var/log/maillog | grep`) সেটি অসম্পূর্ণ — `grep`-এর পর কোনো **প্যাটার্ন** (search term) দেওয়া হয়নি। তবে আপনার দুটি লক্ষ্য বুঝতে পেরেছি:

---

### ১। **Outgoing SMTP server-এর লগ দেখা**

Postfix/Sendmail-এর মতো MTA (Mail Transfer Agent) ব্যবহার করলে, **outgoing mail** সাধারণত নিচের কিছু কীওয়ার্ড দিয়ে শনাক্ত করা যায়:

- `to=<recipient@example.com>`
- `status=sent`
- `relay=...` (যেমন: `relay=smtp.example.com`)
- `postfix/smtp` বা `postfix/qmgr`

#### উদাহরণ:
```bash
grep "status=sent" /var/log/maillog
```

অথবা আরও স্পষ্টভাবে:
```bash
grep "postfix/smtp.*status=sent" /var/log/maillog
```

> 💡 **Note**:  
> - RHEL/CentOS-এ লগ ফাইলের নাম সাধারণত `/var/log/maillog`  
> - Ubuntu/Debian-এ এটি হয় `/var/log/mail.log`

---

### ২। **নির্দিষ্ট sender ও receiver দিয়ে মেইল লগ দেখা**

ধরুন:
- Sender: `sumon@example.com`
- Receiver: `boss@company.com`

#### কমান্ড:
```bash
grep -E "from=<sumon@example\.com>|to=<boss@company\.com>" /var/log/maillog
```

কিন্তু একই মেইলের সম্পূর্ণ লগ চাইলে, **message ID** ব্যবহার করা ভালো। প্রথমে message ID খুঁজুন:

```bash
grep "from=<sumon@example\.com>" /var/log/maillog | grep "to=<boss@company\.com>"
```

আউটপুটে একটি লাইন পাবেন যেখানে থাকবে কিছু এমন:
```
... postfix/qmgr[1234]: ABC123: from=<sumon@example.com>, size=1234, ...
```

এখানে `ABC123` হলো **queue ID**। এখন সম্পূর্ণ লগ দেখুন:

```bash
grep "ABC123" /var/log/maillog
```

এতে ঐ মেইলের সম্পূর্ণ লগ — sender, receiver, relay, status — সব দেখা যাবে।

---

### সহজ স্ক্রিপ্ট (বাংলায় ব্যবহারের জন্য)

```bash
# নির্দিষ্ট sender ও receiver দিয়ে লগ খুঁজুন
SENDER="sumon@example.com"
RECEIVER="boss@company.com"

# প্রথমে queue ID খুঁজুন
QUEUE_ID=$(grep "from=<$SENDER>" /var/log/maillog | grep "to=<$RECEIVER>" | head -1 | awk '{print $6}' | tr -d ':')

# তারপর সম্পূর্ণ লগ দেখুন
if [ -n "$QUEUE_ID" ]; then
    echo "Found message ID: $QUEUE_ID"
    grep "$QUEUE_ID" /var/log/maillog
else
    echo "No matching email found."
fi
```

> ✅ **সতর্কতা**:  
> - `maillog` ফাইলে `from=` এবং `to=` এর আগে **`<`** এবং পরে **`>`** থাকে (যেমন: `from=<user@domain>`), তাই গ্রেপে সেগুলো অবশ্যই দিতে হবে।  
> - ডট (`.`) হলো regex-এ বিশেষ ক্যারেক্টার, তাই escape করতে হয় (`\.`)।

---

আপনি যদি কোনো নির্দিষ্ট MTA (যেমন Postfix, Sendmail, Exim) ব্যবহার করেন, সেটি জানালে আরও নির্দিষ্ট সাহায্য দেওয়া যাবে।

### USER
ওকে আমি আরো বিস্তারিত তথ্য দিচ্ছি, আমি একটি ISP কোম্পানীতে জব করি । এখানে আমরা ইন্টারনেট সার্ভিস প্রভাইডারের পাশাপাশি ইমেইল সার্ভিস প্রাভাইড করি। আমরা ইমেইল সার্ভিস হিসাবে zimbra paid version provide করি। এখন এই zimbra সাথে সাথে barracuda email security gateway এবং ess.barracuda service ও আছে। এবং দেখলাম zimbra ইউজার গুলো যখন কোনো মেইল সেন্ট করে তখন আলাদা একটি সার্ভার হয়ে যাই ওকে outgoing server না কি বলে আমি ঠিক জানি না, আবার relay বলে ।

আমাকে এই বিষয়টা ভালো করে সহজে বোঝাতে পারবেন ধাপে ধাপে? 

### ASSISTANT
অবশ্যই, Sumon! আপনি ISP-তে কাজ করেন এবং Zimbra + Barracuda ব্যবহার করেন — এটা খুব ভালো তথ্য। আপনার প্রশ্নটি খুব গুরুত্বপূর্ণ: **"Zimbra থেকে মেইল পাঠালে আরেকটি সার্ভারে যায় — সেটা কী? Outgoing server নাকি Relay?"**

আমি এটা আপনাকে **ধাপে ধাপে, সহজ ভাষায়** বুঝিয়ে দিচ্ছি — বাংলায় এবং প্রযুক্তিগত শব্দগুলো স্পষ্ট করে।

---

### 🌐 ধাপ ১: সাধারণ মেইল পাঠানোর প্রক্রিয়া (সহজ ভাবে)

ধরুন, আপনার এক ক্লায়েন্ট `user@yourisp.com` থেকে `friend@gmail.com`-এ মেইল পাঠাচ্ছে।

সাধারণত মেইল পাঠানোর জন্য ৩টি জিনিস লাগে:
1. **Mail Client** (যেমন: Outlook, Webmail, Mobile App) → মেইল লেখে
2. **Outgoing Mail Server** → মেইলটি ইন্টারনেটে পাঠায়
3. **Receiving Mail Server** → গন্তব্যে মেইল পৌঁছে দেয়

কিন্তু আপনার ক্ষেত্রে এখানে **অতিরিক্ত একটি সিকিউরিটি লেয়ার** আছে — সেটা হলো **Barracuda Email Security Gateway**।

---

### 🧱 ধাপ ২: আপনার সিস্টেমের আসল ফ্লো (Zimbra + Barracuda)

আপনার সেটআপ এরকম:

```
[Zimbra Mail Server] 
        ↓ (মেইল পাঠায়)
[Barracuda Email Security Gateway] ← এটা আপনার "Outbound Relay"
        ↓ (স্ক্যান করে, স্প্যাম/ভাইরাস চেক করে)
[Internet / Gmail / অন্য মেইল সার্ভার]
```

#### ✅ এখানে:
- **Zimbra** হলো আপনার **মেইল সার্ভার** (যেখানে ইউজারদের মেইলবক্স থাকে)।
- কিন্তু Zimbra সরাসরি ইন্টারনেটে মেইল পাঠায় না।
- বরং, Zimbra **Barracuda-কে মেইল দেয়** — এটাকে বলে **"Outbound Relay"** বা **"Smart Host"**।
- Barracuda মেইলটি **স্ক্যান করে**, তারপর ইন্টারনেটে পাঠায়।

> 🔹 **Relay (রিলে)** মানে হলো: "এক সার্ভার থেকে অন্য সার্ভারে মেইল ফরওয়ার্ড করা"।  
> 🔹 **Outgoing SMTP Server** হলো সেই সার্ভার যে মেইল ইন্টারনেটে পাঠায়। আপনার ক্ষেত্রে **Barracuda-ই আসল outgoing SMTP server**।

---

### 📌 সহজ উদাহরণ:

ধরুন:
- Zimbra IP: `192.168.10.10`
- Barracuda IP: `192.168.10.20`

Zimbra-তে কনফিগার করা থাকবে:
```text
Outbound SMTP Relay = 192.168.10.20 (Barracuda)
Port = 25 or 587
```

ফলে:
- যখন কোনো ইউজার মেইল পাঠায় → Zimbra তা Barracuda-তে পাঠায় → Barracuda তা ইন্টারনেটে পাঠায়।

---

### 📊 লগ কোথায় দেখবেন?

1. **Zimbra লগ**:  
   `/opt/zimbra/log/mailbox.log` বা `/var/log/zimbra.log`  
   → এখানে দেখবেন Zimbra **Barracuda-তে মেইল পাঠিয়েছে কিনা**।

2. **Barracuda লগ**:  
   Barracuda-র ওয়েব ইন্টারফেসে (GUI) → **Message Log** বা **Outbound Log**  
   → এখানে দেখবেন মেইলটি **স্ক্যান হয়েছে কিনা**, **গন্তব্যে পৌঁছেছে কিনা**, **কোনো এরর আছে কিনা**।

> 🔍 আপনি যদি `/var/log/maillog` চেক করেন, সেটা সম্ভবত **Zimbra সার্ভারের লগ** — কিন্তু সেখানে শুধু দেখা যাবে:  
> `"Relaying to 192.168.10.20"`  
> কিন্তু **গন্তব্যে পৌঁছেছে কিনা** — সেটা **Barracuda লগে** দেখতে হবে।

---

### 📝 সংক্ষেপে (Summary):

| প্রশ্ন | উত্তর |
|--------|--------|
| আমার Zimbra থেকে মেইল আলাদা সার্ভারে যায় কেন? | কারণ আপনি **Barracuda** ব্যবহার করছেন সিকিউরিটির জন্য। |
| ওই সার্ভারটি কী? | এটি আপনার **Outbound SMTP Relay** বা **Smart Host**। |
| এটা কি outgoing server? | **হ্যাঁ**, কিন্তু **Zimbra নয় — Barracuda হলো আসল outgoing server**। |
| মেইল লগ কোথায় দেখব? | Zimbra লগে → রিলে হয়েছে কিনা; Barracuda লগে → ইন্টারনেটে পাঠানো হয়েছে কিনা। |

---

### 💡 পরামর্শ:

- যদি কোনো মেইল **পৌঁছায় না**, প্রথমে **Barracuda-র Message Log** চেক করুন।
- Zimbra-তে `zmprov gs yourserver | grep zimbraMtaRelayHost` কমান্ড দিয়ে দেখুন কোন সার্ভারে রিলে করছে।
- Barracuda-তে **"Outbound"** বা **"Sent"** ট্যাবে গেলে সব outgoing মেইল দেখা যাবে — sender, receiver, status সহ।

---

আপনি চাইলে আমি আপনার জন্য **Zimbra + Barracuda আউটবাউন্ড মেইল ট্রাবলশুটিং চেকলিস্ট** বা **লগ ফিল্টারিং কমান্ড** ও দিতে পারি।

আপনার কোন অংশটুকু আরও ক্লিয়ার করতে চান? 😊

### USER
আমার ক্ষেত্রে সেন্ডারঃ sajib@debonairgroupbd.com
আর রিসিভারঃ jing_haidong@teijin.com.cn, yan_fei@teijin.com.cn
অথবা শুধু বিসিভার ডোমেইন দিয়ে সার্চ করতে চাই যে সেই সেন্ট মেইল আসলেই সেন্ট হয়েছে কি না?

### ASSISTANT
ধন্যবাদ, Sumon! আপনার দেওয়া তথ্য অনুযায়ী:

- **Sender**: `sajib@debonairgroupbd.com`  
- **Receivers**:  
  - `jing_haidong@teijin.com.cn`  
  - `yan_fei@teijin.com.cn`  
- অথবা শুধু **`teijin.com.cn`** ডোমেইনে পাঠানো মেইলগুলো চেক করতে চান  
- লক্ষ্য: **নিশ্চিত করা যে মেইলগুলো সফলভাবে পাঠানো (sent) হয়েছে কিনা**

আপনি Zimbra + Barracuda ব্যবহার করেন, তাই আমি আপনাকে **ধাপে ধাপে** বলছি কীভাবে এটা চেক করবেন — **Zimbra লগ** এবং **Barracuda লগ** উভয় জায়গায়।

---

## ✅ ধাপ ১: Zimbra সার্ভারে লগ চেক করুন

Zimbra মেইলটি Barracuda-এ পাঠিয়েছে কিনা সেটা দেখুন।

### 🔍 কমান্ড (Zimbra সার্ভারে SSH করে):
```bash
sudo grep "sajib@debonairgroupbd.com" /var/log/zimbra.log | grep -i "relay"
```

অথবা `teijin.com.cn` ডোমেইনে পাঠানো সব মেইল দেখতে:
```bash
sudo grep "teijin\.com\.cn" /var/log/zimbra.log
```

> 💡 Zimbra-তে আউটবাউন্ড মেইলের লগ সাধারণত `/var/log/zimbra.log` বা `/opt/zimbra/log/mailbox.log`-এ থাকে। কিন্তু **MTA (Postfix)** লগ হয় `/var/log/zimbra.log`-এ।

### 📌 যে লাইনটি খুঁজুন:
```
... postfix/smtp[1234]: ABC123: to=<jing_haidong@teijin.com.cn>, relay=192.168.x.x[192.168.x.x]:25, delay=..., status=sent
```

- `status=sent` → Zimbra **Barracuda-তে মেইল সফলভাবে পাঠিয়েছে**
- `relay=192.168.x.x` → এটা আপনার **Barracuda-র IP**

> ⚠️ কিন্তু! `status=sent` মানে **Barracuda-তে পৌঁছেছে**, কিন্তু **গন্তব্যে (teijin.com.cn) পৌঁছেছে কিনা** — সেটা **Barracuda লগে** দেখতে হবে।

---

## ✅ ধাপ ২: Barracuda Email Security Gateway-এ লগ চেক করুন

এটাই সবচেয়ে গুরুত্বপূর্ণ ধাপ — কারণ Barracuda হলো আপনার **real outgoing SMTP server**।

### 🔧 কীভাবে লগ দেখবেন:
1. Barracuda-র ওয়েব ইন্টারফেসে লগইন করুন (`https://<barracuda-ip>`)
2. যান: **Message Log** → **Outbound** বা **Sent** ট্যাবে
3. সার্চ ফিল্টার দিন:
   - **Sender**: `sajib@debonairgroupbd.com`
   - **Recipient**: `teijin.com.cn` (বা সম্পূর্ণ ইমেইল)
4. অথবা **Advanced Search**-এ লিখুন:
   ```
   recipient_domain:teijin.com.cn AND sender:sajib@debonairgroupbd.com
   ```

### 📌 লগে যা দেখবেন:
| কলাম | মানে |
|------|------|
| **Status** | `Delivered` = সফল, `Deferred` = সাময়িক সমস্যা, `Rejected` = ফেরত |
| **Remote Host** | `teijin.com.cn`-এর MX সার্ভার (যেমন: `mx1.teijin.com`) |
| **Message ID** | Zimbra-র `ABC123`-এর মতো ID (Barracuda-র নিজস্ব ID থাকতে পারে) |

> ✅ যদি **Status = "Delivered"** দেখেন → মেইল **সফলভাবে পাঠানো হয়েছে**।  
> ❌ যদি **"Rejected"** বা **"Bounced"** দেখেন → কারণ (যেমন: SPF fail, IP blacklisted) লগে লেখা থাকবে।

---

## ✅ ধাপ ৩: শুধু `teijin.com.cn` ডোমেইনে পাঠানো সব মেইল দেখুন

### Zimbra-তে (quick check):
```bash
sudo grep -i "to=<.*@teijin\.com\.cn>" /var/log/zimbra.log | grep "status=sent"
```

### Barracuda-তে:
- **Message Log → Outbound**
- **Recipient contains**: `teijin.com.cn`
- **Time range**: আপনার প্রয়োজনীয় সময় (যেমন: গত 24 ঘণ্টা)

---

## 🛠️ বোনাস: কমান্ড লাইনে দ্রুত চেক (Zimbra সার্ভার থেকে)

```bash
# নির্দিষ্ট সেন্ডার + ডোমেইন
grep "sajib@debonairgroupbd.com" /var/log/zimbra.log | grep "teijin\.com\.cn"

# শুধু সেন্ট মেইল
grep "status=sent" /var/log/zimbra.log | grep "teijin\.com\.cn"
```

> 📝 **Note**: ডট (`.`) হলো regex ক্যারেক্টার, তাই `\.` লিখতে হয়।

---

## 📌 চূড়ান্ত উত্তর:

> **হ্যাঁ, আপনি নিশ্চিত করতে পারবেন যে মেইল সেন্ট হয়েছে কিনা — কিন্তু শুধু Zimbra লগ দেখলে হবে না।**  
> **Barracuda-র "Message Log → Outbound" এ গিয়ে `teijin.com.cn` সার্চ করুন।**  
> **যদি Status = "Delivered" দেখেন → মেইল সফলভাবে পাঠানো হয়েছে।**

---

আপনি চাইলে আমি আপনার জন্য একটি **PDF চেকলিস্ট** বা **Google Form-এ রিপোর্ট করার জন্য সংক্ষিপ্ত টেমপ্লেট** ও তৈরি করে দিতে পারি। জানান! 😊

### USER
এটি আমাদের কোম্পানীর barracuda email gateway এর outbound settings.

### ASSISTANT
ধন্যবাদ, Sumon! আপনি যে Barracuda Email Security Gateway-এর **Outbound Settings** এর স্ক্রিনশট দিয়েছেন — এটা খুবই গুরুত্বপূর্ণ। এখান থেকে আমরা পরিষ্কারভাবে বুঝতে পারছি:

---

## 🔍 স্ক্রিনশট থেকে প্রাপ্ত তথ্য (সহজ ভাষায়):

### 1. **Outbound SMTP Host (Smart Host)**:
```
IP: 45.64.135.186
Port: 25
```

> ✅ এটি হলো আপনার Barracuda-র **আসল আউটগোয়িং মেইল সার্ভার** — অর্থাৎ, Zimbra এই IP-তে মেইল পাঠায়, আর Barracuda এই IP-কে **relay** হিসেবে ব্যবহার করে।

➡️ এটি **external mail server** — সম্ভবত আপনার ISP বা থার্ড পার্টি মেইল সার্ভিস (যেমন: ESS.Barracuda, SendGrid, Amazon SES, বা অন্য কোনো Smart Host)।

---

### 2. **Authentication (Username & Password)**:
```
Username: admin
Password: **********
```

> ⚠️ এটি বলে যে, Barracuda **এই external SMTP server-এ লগইন করে** মেইল পাঠায়।  
> এটি **SMTP AUTH** ব্যবহার করে — মানে শুধু IP নয়, ইউজারনেম/পাসওয়ার্ড দিয়ে অথেন্টিকেশন করে।

---

### 3. **Force TLS: No**

> ❌ TLS (Secure Connection) **বাধ্যতামূলক নয়**।  
> অর্থাৎ, মেইল পাঠানোর সময় **encrypted connection** ব্যবহার করা হচ্ছে না (যদি external server না জোর করে)।  
> 📌 **সুপারিশ**: যদি external server (45.64.135.186) TLS সাপোর্ট করে, তাহলে `Force TLS: Yes` সেট করুন — নিরাপত্তা বাড়বে।

---

### 4. **Relay Using Authentication → SASL/SMTP Authentication: Off**

> ✅ এটি বলে যে, Barracuda **অন্য কোনো মেইল সার্ভারকে relay করার জন্য authentication ব্যবহার করে না** — কারণ এখানে আপনি **external SMTP host**-এ মেইল পাঠাচ্ছেন, যেখানে আপনি ইতিমধ্যে `admin` দিয়ে অথেন্টিকেট করছেন।

---

### 5. **Use Destination Mail Server as SMTP AUTH Proxy: Yes**

> ✅ এটি বলে যে, যদি কোনো মেইল **destination server** (যেমন: teijin.com.cn) নিজেই SMTP AUTH চায়, তাহলে Barracuda সেটি ব্যবহার করবে।  
> কিন্তু এটি সাধারণত কাজে লাগে না — কারণ বেশিরভাগ গন্তব্য সার্ভার (Gmail, Outlook, Teijin) আপনার থেকে অথেন্টিকেশন চায় না — শুধু SPF/DKIM/DMARC চেক করে।

---

## 📌 সারাংশ (Summary):

| প্যারামিটার | মান | ব্যাখ্যা |
|--------------|------|----------|
| **Outbound SMTP Host** | `45.64.135.186:25` | আপনার Barracuda এই IP-তে মেইল পাঠায় — এটি আসল "outgoing server" |
| **Authentication** | `admin / password` | Barracuda এই ক্রেডেনশিয়াল দিয়ে external server-এ লগইন করে |
| **Force TLS** | `No` | সেকিউর কানেকশন বাধ্যতামূলক নয় — নিরাপত্তা ঝুঁকি থাকতে পারে |
| **SASL/SMTP Auth** | `Off` | Barracuda অন্য কাউকে relay করতে অথেন্টিকেশন ব্যবহার করে না |
| **Destination Auth Proxy** | `Yes` | গন্তব্য সার্ভার যদি অথেন্টিকেশন চায়, তাহলে ব্যবহার করে |

---

## 💡 পরামর্শ (Troubleshooting & Best Practice):

1. **TLS চালু করুন (যদি সম্ভব)**  
   - `Force TLS: Yes` সেট করুন — যদি `45.64.135.186` TLS সাপোর্ট করে।  
   - নিরাপত্তা বাড়বে, এবং কিছু মেইল সার্ভার (যেমন: Gmail) TLS ছাড়া মেইল নিতে অস্বীকার করতে পারে।

2. **External SMTP server-এর সাথে সংযোগ চেক করুন**  
   ```bash
   telnet 45.64.135.186 25
   ```
   যদি কানেকশন হয় এবং `220` রেসপন্স আসে — তাহলে সার্ভার অ্যাক্টিভ।

3. **Barracuda-র Outbound Log-এ গিয়ে দেখুন**:  
   - মেইল কোন স্ট্যাটাসে গেছে? (`Delivered`, `Deferred`, `Rejected`)  
   - `teijin.com.cn` ডোমেইনে পাঠানো মেইলগুলো সফল হয়েছে কিনা?

4. **Zimbra-তে Relay Host চেক করুন**  
   ```bash
   zmprov gs yourserver | grep zimbraMtaRelayHost
   ```
   এটি দেখাবে যে Zimbra কোন IP-তে মেইল পাঠাচ্ছে — যেটা Barracuda-র IP হওয়া উচিত (যেমন: `192.168.x.x`)।

---

## 📌 আপনার প্রশ্নের উত্তর:

> **“sajib@debonairgroupbd.com” থেকে “teijin.com.cn”-এ মেইল সেন্ট হয়েছে কি না?”**

✅ **যেভাবে চেক করবেন:**

1. **Barracuda GUI → Message Log → Outbound**  
   - Filter: `Recipient contains: teijin.com.cn`  
   - Filter: `Sender: sajib@debonairgroupbd.com`  
   - দেখুন: **Status = Delivered** কিনা

2. **Zimbra লগে দেখুন**:  
   ```bash
   grep "sajib@debonairgroupbd.com" /var/log/zimbra.log | grep "teijin\.com\.cn"
   ```
   → `status=sent` থাকলে মেইল Barracuda-তে পৌঁছেছে।

3. **Barracuda-র লগে `45.64.135.186`-এ মেইল পাঠানোর রেকর্ড দেখুন** — যদি `Delivered` থাকে → মেইল সফলভাবে পাঠানো হয়েছে।

---

আপনি চাইলে আমি আপনার জন্য **একটি সহজ PDF চেকলিস্ট** বা **Barracuda লগ ফিল্টারিং কমান্ড** ও তৈরি করে দিতে পারি। জানান! 😊

### USER

এটি আমাদের নিজস্ব আউটগোয়িং মেইল সার্ভার 45.64.135.186  — অর্থাৎ, Zimbra এই IP-তে মেইল পাঠায়, আর Barracuda এই IP-কে relay হিসেবে ব্যবহার করে, এই সার্ভার চেক করা দেখান যে এই আউটগোয়িং মেইল সার্ভারে কি কি ইন্সটল করা আছে এবং কিভাবে কি চেক করতে হবে তারপরে  এই এই আউটগোয়িং মেইল সার্ভারকে ম্যানেজ করার সম্পূর্ণ গাইডলাইন দিন ধাপে-ধাপে?



### ASSISTANT
অবশ্যই, Sumon! আপনি বলেছেন:

> **“45.64.135.186 আমাদের নিজস্ব আউটগোয়িং মেইল সার্ভার”**  
> → অর্থাৎ, এটি **আপনার কোম্পানির মালিকানাধীন SMTP relay server**, যেখানে **Barracuda** মেইল পাঠায়, আর সেখান থেকে মেইল ইন্টারনেটে যায়।

আপনি চাচ্ছেন:
1. **এই সার্ভারে কী কী ইনস্টল আছে তা চেক করার উপায়**  
2. **এই সার্ভার ম্যানেজ করার সম্পূর্ণ ধাপে-ধাপে গাইডলাইন**

আমি আপনাকে **বাস্তব প্রোডাকশন এনভায়রনমেন্টের জন্য** একটি সম্পূর্ণ, নিরাপদ ও ব্যবহারযোগ্য গাইড দিচ্ছি — ধাপে ধাপে।

---

## 🔐 প্রাথমিক ধাপ: সার্ভারে SSH অ্যাক্সেস নিশ্চিত করুন

আপনার প্রথম কাজ হবে **45.64.135.186 সার্ভারে SSH লগইন** করা।

```bash
ssh admin@45.64.135.186
# বা
ssh root@45.64.135.186
```

> ✅ নোট: আপনি ISP-এ কাজ করেন, তাই এই সার্ভারটি সম্ভবত **Linux (Ubuntu/CentOS)** এ চলছে।

---

## 🧪 ধাপ ১: সার্ভারে কী কী ইনস্টল আছে তা চেক করুন

### 1.1. **অপারেটিং সিস্টেম চেক করুন**
```bash
cat /etc/os-release
```
→ দেখবেন: Ubuntu, CentOS, Debian ইত্যাদি।

---

### 1.2. **মেইল সার্ভার (MTA) চেক করুন**

সাধারণত আউটগোয়িং মেইল সার্ভারে **Postfix**, **Sendmail**, বা **Exim** থাকে।

```bash
# Postfix চলছে কিনা?
systemctl status postfix

# Sendmail?
systemctl status sendmail

# Exim?
systemctl status exim4   # Debian/Ubuntu
systemctl status exim    # CentOS

# কোন MTA ইনস্টল আছে তা দেখুন:
ps aux | grep -E 'postfix|sendmail|exim'
```

> 💡 সম্ভাব্য ফলাফল:  
> - যদি `postfix` চলছে → এটি **Postfix MTA**  
> - যদি কিছু না পান → সম্ভবত **Zimbra বা অন্য কিছু** চলছে

---

### 1.3. **পোর্ট 25 (SMTP) লিসেন করছে কিনা চেক করুন**
```bash
ss -tulnp | grep :25
# বা
netstat -tulnp | grep :25
```

আউটপুট যদি হয়:
```
tcp 0 0 0.0.0.0:25 0.0.0.0:* LISTEN 1234/postfix
```
→ মানে **Postfix পোর্ট 25-এ চলছে**।

---

### 1.4. **ফায়ারওয়াল চেক করুন**
```bash
# Ubuntu/Debian
sudo ufw status

# CentOS/RHEL
sudo firewall-cmd --list-all
# বা
sudo iptables -L
```

→ নিশ্চিত করুন যে **পোর্ট 25 (SMTP)** খোলা আছে **Barracuda IP** থেকে (যেমন: `192.168.x.x` বা আপনার internal IP)।

---

### 1.5. **কোন প্যাকেজ ইনস্টল আছে তা দেখুন**
```bash
# Ubuntu/Debian
dpkg -l | grep -E 'postfix|mail|smtp'

# CentOS/RHEL
rpm -qa | grep -E 'postfix|sendmail|mail'
```

---

## 🛠️ ধাপ ২: মেইল সার্ভার কনফিগারেশন চেক করুন

### 2.1. **Postfix কনফিগারেশন (যদি Postfix ব্যবহার করেন)**
```bash
# মূল কনফিগ ফাইল
cat /etc/postfix/main.cf

# গুরুত্বপূর্ণ সেটিংস চেক করুন:
postconf myhostname
postconf mydomain
postconf myorigin
postconf inet_interfaces
postconf mynetworks          # কোন IP থেকে relay অনুমোদিত?
postconf relayhost
postconf smtpd_recipient_restrictions
```

> ✅ **`mynetworks`** এ অবশ্যই **Barracuda-র IP** (যেমন: `192.168.10.20`) থাকতে হবে — নাহলে Barracuda মেইল পাঠাতে পারবে না।

উদাহরণ:
```ini
mynetworks = 127.0.0.0/8, 192.168.10.0/24
```

---

### 2.2. **লগ ফাইল চেক করুন**
```bash
# Ubuntu/Debian
tail -f /var/log/mail.log

# CentOS/RHEL
tail -f /var/log/maillog
```

→ এখানে দেখুন:  
- Barracuda থেকে মেইল আসছে কিনা  
- মেইল গন্তব্যে পৌঁছাচ্ছে কিনা (`status=sent`)  
- কোনো এরর আছে কিনা (`reject`, `defer`, `timeout`)

---

## 📦 ধাপ ৩: প্রয়োজনীয় সফটওয়্যার ইনস্টল/আপডেট করুন

### 3.1. **Postfix ইনস্টল (যদি না থাকে)**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postfix mailutils

# CentOS/RHEL
sudo yum install postfix
sudo systemctl enable --now postfix
```

> 📌 ইনস্টলেশন সময় **"Internet Site"** সিলেক্ট করুন।

---

### 3.2. **SPF, DKIM, DMARC সেটআপ করুন (অত্যন্ত গুরুত্বপূর্ণ!)**

#### ✅ SPF রেকর্ড (DNS-এ যোগ করুন):
```dns
debonairgroupbd.com.  IN TXT "v=spf1 ip4:45.64.135.186 -all"
```

#### ✅ DKIM (OpenDKIM ইনস্টল করুন):
```bash
# Ubuntu
sudo apt install opendkim opendkim-tools

# কনফিগার করুন (আমি পরে ডিটেইলস দিচ্ছি)
```

#### ✅ DMARC (DNS-এ):
```dns
_dmarc.debonairgroupbd.com.  IN TXT "v=DMARC1; p=quarantine; rua=mailto:admin@debonairgroupbd.com"
```

> 🔥 না হলে আপনার মেইল **Gmail, Outlook, Teijin**-এ **স্প্যাম** বা **রিজেক্ট** হবে!

---

## 🛡️ ধাপ ৪: নিরাপত্তা ও মনিটরিং

### 4.1. **Open Relay চেক করুন (অত্যন্ত জরুরি!)**
আপনার সার্ভার যেন **কেউ মিসইউজ করতে না পারে**।

অনলাইন টুল দিয়ে চেক করুন:
- https://mxtoolbox.com/diagnostic.aspx
- https://www.mail-tester.com

অথবা কমান্ড লাইন থেকে:
```bash
telnet 45.64.135.186 25
> EHLO test.com
> MAIL FROM: <test@test.com>
> RCPT TO: <someone@gmail.com>
```
→ যদি `250 OK` আসে → **Open Relay!** (খুব বিপজ্জনক)

✅ সমাধান: `main.cf`-এ নিশ্চিত করুন:
```ini
smtpd_recipient_restrictions =
    permit_mynetworks,
    reject_unauth_destination
```

---

### 4.2. **লগ রোটেশন ও মনিটরিং**
```bash
# Logrotate চেক করুন
cat /etc/logrotate.d/mail

# Fail2ban ইনস্টল করুন (SMTP brute-force প্রতিরোধ)
sudo apt install fail2ban
```

---

### 4.3. **ব্যাকআপ নিন**
```bash
# Postfix কনফিগ ব্যাকআপ
sudo cp -r /etc/postfix /backup/postfix-$(date +%F)
```

---

## 📋 ধাপ ৫: মেইল সার্ভার ম্যানেজমেন্ট চেকলিস্ট (Daily/Weekly)

| কাজ | কমান্ড / পদ্ধতি |
|------|------------------|
| মেইল লগ চেক | `tail -f /var/log/mail.log` |
| সার্ভার লোড চেক | `htop`, `df -h`, `free -m` |
| ডিস্ক স্পেস চেক | `df -h /var` |
| SPF/DKIM ভেরিফাই | https://dmarcian.com/spf-survey/ |
| ব্ল্যাকলিস্ট চেক | https://mxtoolbox.com/blacklists.aspx |
| সার্ভিস রিস্টার্ট | `sudo systemctl restart postfix` |
| কনফিগ ব্যাকআপ | `tar -czf postfix-backup.tar.gz /etc/postfix` |

---

## 📝 সারাংশ: আপনার আউটগোয়িং মেইল সার্ভার ম্যানেজমেন্ট গাইড

1. **SSH করুন** → `45.64.135.186`
2. **MTA চেক করুন** → `postfix` বা অন্য কিছু?
3. **`mynetworks` সেট করুন** → Barracuda IP allow করুন
4. **SPF/DKIM/DMARC সেটআপ করুন** → না হলে মেইল ডেলিভারি ফেইল হবে
5. **Open Relay চেক করুন** → নিরাপত্তা জরুরি
6. **লগ মনিটর করুন** → `/var/log/mail.log`
7. **ব্ল্যাকলিস্ট চেক করুন** → MXToolbox দিয়ে নিয়মিত

---

## 💡 পরামর্শ:

- আপনার ক্ষেত্রে, যেহেতু **Zimbra → Barracuda → 45.64.135.186 → Internet**,  
  তাই **45.64.135.186**-এর **IP reputation** খুব গুরুত্বপূর্ণ।  
  → নিয়মিত **Spamhaus, Barracuda Reputation** চেক করুন।

- যদি সম্ভব হয়, **Port 587 + TLS** ব্যবহার করুন (Port 25 অনেক সময় ব্লক থাকে)।

---
