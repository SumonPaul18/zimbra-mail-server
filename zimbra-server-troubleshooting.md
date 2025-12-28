# Troubleshooting Zimbra Mail Server

### References:
- Zimbra Incoming Mail Problems
> https://wiki.zimbra.com/wiki/Incoming_Mail_Problems
- Zimbra Mail Routing Problem:
> https://wiki.zimbra.com/wiki/Mail_Routing_Issues

---

#### View Zimbra Logs
```
tail -f /var/log/zimbra.log
```
#### Verify zimbra status
```
su zimbra
zmcontrol status
```
#### Restart zimbra Services
```
su zimbra
zmcontrol restart
```
---


## 1. Track messages sent and received by a user:

shows the logs for all the users:
```
/opt/zimbra/libexec/zmmsgtrace      
```
Using '-s' shows all the emails sent by specefic user.
```
/opt/zimbra/libexec/zmmsgtrace -s sumon@iotlogy.xyz       
```
Using '-r' sorts emails by the receiver. So for the emails sent to 'gmail.com'.
```
/opt/zimbra/libexec/zmmsgtrace -r '@gmail.com'         
```
---


## 2. এই ত্রুটিটি (**"system failure: exception during auth {RemoteManager: mail.pedrollobd.com->zimbra@mail.pedrollobd.com:22}"**) Zimbra মেইল সার্ভারে **SSH-ভিত্তিক রিমোট ম্যানেজমেন্ট** সংক্রান্ত একটি সমস্যা নির্দেশ করছে।

### ব্যাখ্যা:

Zimbra সার্ভার নিজেকে ম্যানেজ করার জন্য অভ্যন্তরীণভাবে **SSH** ব্যবহার করে — যেমন: সার্ভিস রিস্টার্ট, কনফিগারেশন আপডেট ইত্যাদি।  
এই ত্রুটি বলছে যে, Zimbra যখন `zimbra` ইউজার হিসেবে `mail.pedrollobd.com` সার্ভারে **SSH দিয়ে লগইন** করার চেষ্টা করছিল (পোর্ট 22 এ), তখন **অথেন্টিকেশন ব্যর্থ** হয়েছে।

---

### সম্ভাব্য কারণগুলো:

1. **`zimbra` ইউজারের SSH key নষ্ট বা মিসিং**  
   - Zimbra সাধারণত `~zimbra/.ssh/authorized_keys` এবং `~zimbra/.ssh/id_rsa` ফাইল ব্যবহার করে localhost-এ SSH করে।
   - যদি `id_rsa` (private key) বা `authorized_keys` (public key) ফাইল মিসিং/করাপ্ট হয়, তাহলে এই ত্রুটি আসে।

2. **SSH সার্ভিস বন্ধ বা কনফিগারেশন সমস্যা**  
   - `sshd` সার্ভিস চালু আছে কিনা চেক করুন।
   - `/etc/ssh/sshd_config` ফাইলে `PermitRootLogin`, `PubkeyAuthentication`, `AuthorizedKeysFile` ইত্যাদি সেটিংস ঠিক আছে কিনা দেখুন।
   - নিশ্চিত করুন যে `zimbra` ইউজারের জন্য SSH login অনুমোদিত।

3. **ফাইল পারমিশন সমস্যা**  
   - `~zimbra/.ssh` ডিরেক্টরির permission **700** হতে হবে।
   - `~zimbra/.ssh/authorized_keys` এর permission **600** হতে হবে।
   - মালিকানা (`chown`) `zimbra:zimbra` হতে হবে।

4. **SELinux বা AppArmor ব্লক করছে** (যদি RHEL/CentOS/Ubuntu ব্যবহার করেন)

5. **Zimbra কনফিগারেশনে হোস্টনেম মিসম্যাচ**  
   - `zmhostname` কমান্ড দিয়ে চেক করুন যে Zimbra ঠিক হোস্টনেম (`mail.pedrollobd.com`) ব্যবহার করছে কিনা।
   - `/etc/hosts` ফাইলে `127.0.0.1 mail.pedrollobd.com` এন্ট্রি থাকা উচিত।

---

### সমাধানের ধাপসমূহ:

#### ✅ 1. Zimbra SSH key রিজেনারেট করুন:
```bash
su - zimbra
zmsshkeygen
zmupdateauthkeys
```

#### ✅ 2. SSH লগইন টেস্ট করুন:
```bash
su - zimbra
ssh -i .ssh/id_rsa zimbra@mail.pedrollobd.com
```
> যদি পাসওয়ার্ড চায় বা "Permission denied (publickey)" দেখায়, তাহলে key setup ঠিক নেই।

#### ✅ 3. পারমিশন চেক করুন:
```bash
ls -ld ~zimbra/.ssh
ls -l ~zimbra/.ssh/authorized_keys
```
- `.ssh` → `drwx------` (700)
- `authorized_keys` → `-rw-------` (600)

#### ✅ 4. SSH সার্ভিস চালু আছে কিনা:
```bash
systemctl status sshd
```

#### ✅ 5. Zimbra হোস্টনেম ও `/etc/hosts` চেক:
```bash
zmhostname
cat /etc/hosts
```
অবশ্যই নিশ্চিত করুন:
```
127.0.0.1   localhost localhost.localdomain
192.168.x.x mail.pedrollobd.com mail   # (আপনার সার্ভারের আসল IP)
```

---

### সতর্কতা:
- এই সমস্যা থাকলে Zimbra Admin UI কাজ করবে না, মেইল ডেলিভারি বা কোনো অ্যাডমিন অপারেশন ব্যর্থ হবে।
- সমাধান না হলে **Zimbra রিস্টার্ট** (`zmcontrol restart`) করার আগে key ঠিক করুন।

---

## 3. 📧 Zimbra সার্ভারের TLS/SSL ত্রুটি সমাধান গাইড

### 🛑 সমস্যা এবং ত্রুটির কারণ

আপনার Zimbra সার্ভারে `zmcontrol status` বা `zmcontrol start` চালানোর সময় যে ত্রুটিটি আসছে, তা হলো:

> `Unable to start TLS: SSL connect attempt failed error:14090086:SSL routines:ssl3_get_server_certificate:certificate verify failed when connecting to ldap master.`

এই ত্রুটিটির অর্থ হলো:

  * **Zimbra-এর পরিষেবাগুলি** (যেমন Mailboxd) **LDAP মাস্টার সার্ভারের** (যা ব্যবহারকারীর তথ্য সংরক্ষণ করে) সাথে সংযোগ স্থাপন করার চেষ্টা করছে।
  * এই সংযোগটি **TLS/SSL এনক্রিপশনের** মাধ্যমে সুরক্ষিত।
  * LDAP মাস্টার সার্ভার যে **SSL সার্টিফিকেটটি** প্রদান করছে, ক্লায়েন্ট সেটি **যাচাই (verify) করতে পারছে না** বা সেটিকে **অবিশ্বাসযোগ্য (untrusted)** মনে করছে।

#### **মূল কারণসমূহ**

1.  **সার্টিফিকেট মেয়াদোত্তীর্ণ (Expired):** Zimbra দ্বারা ব্যবহৃত **স্ব-স্বাক্ষরিত (self-signed) SSL সার্টিফিকেটটির** মেয়াদ শেষ হয়ে গেছে।
2.  **CA চেইনের সমস্যা:** আপনি যদি একটি বাণিজ্যিক সার্টিফিকেট ব্যবহার করেন, তবে **ইন্টারমিডিয়েট সার্টিফিকেট অথরিটি (CA) চেইনটি** সঠিকভাবে ইনস্টল করা হয়নি বা এটিতে কোনো ত্রুটি রয়েছে।
3.  **সময় অসঙ্গতি (Time Skew):** সার্ভার এবং ক্লায়েন্টের ঘড়ির সময়ের মধ্যে উল্লেখযোগ্য পার্থক্য রয়েছে, যার ফলে সার্টিফিকেট যাচাই ব্যর্থ হচ্ছে।

-----

### 🚀 দ্রুত সাময়িক সমাধান (Workaround)

পরিষেবাগুলিকে দ্রুত চালু করার জন্য আপনি **অস্থায়ীভাবে LDAP-এর জন্য TLS/SSL বাধ্যতামূলকতা নিষ্ক্রিয়** করতে পারেন। এটি বিশেষ করে একক-সার্ভার সেটআপের জন্য নিরাপদ, কারণ সংযোগটি সার্ভারের অভ্যন্তরেই ঘটছে।

**দ্রষ্টব্য:** এই কমান্ডগুলি চালানোর জন্য আপনাকে অবশ্যই `zimbra` ব্যবহারকারী হিসেবে লগইন করতে হবে।

1.  **Zimbra ব্যবহারকারী হিসেবে লগইন করুন:**

    ```bash
    su - zimbra
    ```

2.  **TLS/SSL বাধ্যতামূলকতা নিষ্ক্রিয় করুন:**
    নিচের দুটি কনফিগারেশন প্যারামিটার পরিবর্তন করুন:

    ```bash
    zmlocalconfig -e ldap_starttls_required=false
    zmlocalconfig -e ldap_starttls_supported=0
    ```

3.  **Zimbra পরিষেবা চালু করুন:**

    ```bash
    zmcontrol start
    ```

4.  **অবস্থা যাচাই করুন:**

    ```bash
    zmcontrol status
    ```

    এবার পরিষেবাগুলি কোনো SSL ত্রুটি ছাড়াই চালু হওয়া উচিত।

-----

### ✅ স্থায়ী সমাধান (Permanent Fix)

স্থায়ীভাবে এই সমস্যা সমাধানের জন্য আপনাকে অবশ্যই একটি **বৈধ এবং স্বীকৃত SSL সার্টিফিকেট** স্থাপন করতে হবে।

#### **ধাপ ১: বর্তমান সার্টিফিকেট স্ট্যাটাস পরীক্ষা করা**

```bash
/opt/zimbra/bin/zmcertmgr viewcrts
```

এই কমান্ডটি আপনাকে দেখাবে কোন সার্টিফিকেটটি ব্যবহার করা হচ্ছে, সেটির মেয়াদ কবে শেষ হবে (`Not After` তারিখটি দেখুন) এবং সেটি **স্থাপিত (deployed)** হয়েছে কিনা।

#### **ধাপ ২: সার্টিফিকেট নবায়ন বা পুনরায় স্থাপন**

**১. যদি স্ব-স্বাক্ষরিত (Self-Signed) সার্টিফিকেট ব্যবহার করেন:**
সাধারণত এই ধরনের ত্রুটি স্ব-স্বাক্ষরিত সার্টিফিকেটের মেয়াদ শেষ হলেই হয়। আপনি এটি পুনরায় তৈরি ও স্থাপন করতে পারেন:


#### নতুন CA (Certificate Authority) তৈরি করুন
```
/opt/zimbra/bin/zmcertmgr createca -new
```
#### নতুন CA স্থাপন করুন
```
/opt/zimbra/bin/zmcertmgr deployca
```
#### নতুন সার্টিফিকেট (10 বছরের জন্য) তৈরি করুন
```
/opt/zimbra/bin/zmcertmgr createcrt -new -days 3650
```
#### নতুন সার্টিফিকেট স্থাপন করুন
```
/opt/zimbra/bin/zmcertmgr deploycrt self
```
#### পরিবর্তন কার্যকর করতে পরিষেবা পুনরায় চালু করুন
```
zmcontrol restart
```

**২. যদি বাণিজ্যিক (Commercial) সার্টিফিকেট ব্যবহার করেন:**
যদি আপনার বাণিজ্যিক সার্টিফিকেটটির মেয়াদ শেষ হয়ে যায়, তবে আপনার প্রদানকারীর কাছ থেকে নতুন সার্টিফিকেট ফাইলগুলি ডাউনলোড করুন এবং Zimbra ডকুমেন্টেশন অনুযায়ী তা সঠিকভাবে স্থাপন করুন।

#### **ধাপ ৩: LDAP TLS/SSL পুনরায় সক্রিয় করা (গুরুত্বপূর্ণ)**

স্থায়ীভাবে সমস্যা সমাধানের পর এবং নতুন সার্টিফিকেট স্থাপনের পর, সুরক্ষা নিশ্চিত করার জন্য **TLS/SSL বাধ্যতামূলকতা আবার সক্রিয়** করে দেওয়া উচিত:

```
zmlocalconfig -e ldap_starttls_required=true
zmlocalconfig -e ldap_starttls_supported=1
```
#### পরিষেবা পুনরায় চালু করে পরিবর্তনগুলি কার্যকর করুন
```
zmcontrol restart
```

আপনার Zimbra সার্ভারের সমস্যা ঠিক হয়েছে কিনা, তা নিশ্চিত করতে `zmcontrol status` দিয়ে পরীক্ষা করুন।

---
## আপনার Zimbra সার্ভারে **`/opt/zimbra`** পার্টিশন **100% ফুল** — এটাই মূল সমস্যা।  

```
[zimbra@mail ~]$ zmcontrol status
Host mail.reverie-bd.com
        amavis                  Running
        antispam                Running
        antivirus               Stopped
        zmclamdctl is not running
        zmfreshclamctl is not running
        ldap                    Running
        logger                  Running
        mailbox                 Running
        mta                     Stopped
        zmsaslauthdctl is not running
        postfix is not running
        opendkim                Stopped
        zmopendkimctl is not running.
        service webapp          Running
        snmp                    Stopped
        zmswatch is not running.
        spell                   Stopped
        zmapachectl is not running
        stats                   Stopped
        zimbra webapp           Running
        zimbraAdmin webapp      Running
        zimlet webapp           Running
        zmconfigd               Running

[zimbra@mail ~]$ df -h
Filesystem                     Size  Used Avail Use% Mounted on
devtmpfs                       7.8G     0  7.8G   0% /dev
tmpfs                          7.8G     0  7.8G   0% /dev/shm
tmpfs                          7.8G  769M  7.1G  10% /run
tmpfs                          7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/mapper/centos-root         20G  2.7G   18G  14% /
/dev/sda1                     1014M  194M  821M  20% /boot
/dev/mapper/centos-home         50G  2.8G   48G   6% /var
/dev/mapper/centos-opt_zimbra  945G  937G  8.4G 100% /opt/zimbra
tmpfs                          1.6G     0  1.6G   0% /run/user/0
[zimbra@mail ~]$


```

### আপনার Zimbra সার্ভারে **`/opt/zimbra`** পার্টিশন **100% ফুল** — এটাই মূল সমস্যা।  
Zimbra-র অ্যান্টিভাইরাস (ClamAV), MTA (Postfix), OpenDKIM, SNMP, Spell, Stats ইত্যাদি সার্ভিসগুলো বন্ধ হয়ে গেছে, কারণ সিস্টেমে **ডিস্ক স্পেস নেই**। Zimbra এমন অবস্থায় কোনো লগ বা টেম্প ফাইল লিখতে পারে না, ফলে সার্ভিসগুলো ক্র্যাশ করে।

---

### ✅ সমাধানের ধাপসমূহ (Step-by-step):

#### **ধাপ 1: জরুরি ক্লিনআপ — অপ্রয়োজনীয় ফাইল মুছুন**

1. **Zimbra লগ ফাইলগুলো পরীক্ষা করুন** (সবচেয়ে বড় জায়গা নেয়):
   ```bash
   du -sh /opt/zimbra/log/*
   du -sh /opt/zimbra/mailboxd/logs/*
   ```
   পুরনো লগ ফাইল (যেমন: `mailboxd.out.1`, `zimbra.log.1`) মুছুন:
   ```bash
   sudo su - zimbra
   rm -f /opt/zimbra/log/*.log.* /opt/zimbra/log/*.out.*
   rm -f /opt/zimbra/mailboxd/logs/*.log.*
   ```

2. **ClamAV ভাইরাস ডেটাবেস ক্যাশে চেক করুন**:
   ```bash
   du -sh /opt/zimbra/data/clamav/db/
   ```
   যদি সেখানে পুরনো/করাপ্টেড ডেটাবেস থাকে (যেমন `.cld`, `.cvd` ফাইল), Zimbra রিস্টার্টের পর অটো ডাউনলোড হবে — তাই অস্থায়ীভাবে মুছে ফেলতে পারেন:
   ```bash
   rm -f /opt/zimbra/data/clamav/db/*.cld /opt/zimbra/data/clamav/db/*.cvd
   ```

3. **Zimbra টেম্প ফাইল ক্লিন করুন**:
   ```bash
   rm -rf /opt/zimbra/data/tmp/*
   ```

4. **Core dump বা ক্র্যাশ ফাইল আছে কিনা চেক করুন**:
   ```bash
   find /opt/zimbra -name "core.*" -delete
   ```

---

#### **ধাপ 2: ডিস্ক স্পেস ফ্রি করার পর Zimbra সার্ভিসগুলো রিস্টার্ট করুন**

```bash
su - zimbra
zmcontrol restart
```

> ⚠️ **দ্রষ্টব্য**: `zmcontrol start` না দিয়ে `restart` দিন, কারণ কিছু সার্ভিস আধা-চালু থাকতে পারে।

---

#### **ধাপ 3: ভবিষ্যতে এড়ানোর জন্য — অটো লগ রোটেশন ও মনিটরিং সেট করুন**

1. **লগরোটেট কনফিগার করুন** (যদি না থাকে):
   - `/etc/logrotate.d/zimbra` ফাইলটি চেক করুন।
   - যদি না থাকে, Zimbra-র ডিফল্ট লগরোটেট স্ক্রিপ্ট ইনস্টল করুন:
     ```bash
     /opt/zimbra/libexec/zmfixperms --extended
     /opt/zimbra/bin/zmlogprocess --rotate
     ```

2. **Zimbra-র অটো ক্লিনআপ স্ক্রিপ্ট** (যেমন `zmlogswatchctl`) চালু রাখুন — এটি সিস্টেমের উপর নির্ভর করে।

3. **ডিস্ক মনিটরিং যোগ করুন** (Zabbix/Nagios/ওয়েব UI-তে Alert সেট করুন)।

---

#### **ধাপ 4 (অপশনাল): পার্টিশন সাইজ বাড়ান (দীর্ঘমেয়াদী সমাধান)**

আপনার `/opt/zimbra` পার্টিশনটি **945G-এর মধ্যে 937G ব্যবহৃত** — শুধু ক্লিনআপ করলে সমস্যা আবার ফিরে আসবে।

- **LVM ব্যবহার করে পার্টিশন বাড়ানোর চেষ্টা করুন** (যদি অতিরিক্ত ডিস্ক স্পেস থাকে):
  ```bash
  lvextend -L +100G /dev/mapper/centos-opt_zimbra
  xfs_growfs /opt/zimbra   # যদি XFS ফাইলসিস্টেম হয়
  ```
- অথবা **Zimbra-র মেইলস্টোর (`/opt/zimbra/store`) আলাদা ডিস্কে মাউন্ট করুন**।

---

### 📌 সারাংশ:
- **অবিলম্বে `/opt/zimbra` থেকে কমপক্ষে 5–10 GB জায়গা ফাঁকা করুন**।
- **সার্ভিসগুলো রিস্টার্ট করুন**।
- **লগ ম্যানেজমেন্ট ও ডিস্ক মনিটরিং সেট করুন**।
- **দীর্ঘমেয়াদে ডিস্ক ক্যাপাসিটি প্ল্যানিং করুন**।

যদি ক্লিনআপের পরেও `zmcontrol status`-এ একই সমস্যা থাকে, তাহলে ClamAV বা Postfix-এর লগ চেক করুন:
```bash
tail -f /opt/zimbra/log/clamd.log
tail -f /opt/zimbra/log/mailbox.log
```

---
