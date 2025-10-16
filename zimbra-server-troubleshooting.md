
## Track messages sent and received by a user:

shows the logs for all the users:

/opt/zimbra/libexec/zmmsgtrace      

Using '-s' shows all the emails sent by specefic user.

/opt/zimbra/libexec/zmmsgtrace -s sumon@iotlogy.xyz       

Using '-r' sorts emails by the receiver. So for the emails sent to 'gmail.com'.

/opt/zimbra/libexec/zmmsgtrace -r '@gmail.com'         

...........................

# Troubleshooting incoming mail problems
tail -f /var/log/zimbra.log
ref: https://wiki.zimbra.com/wiki/Incoming_Mail_Problems

## Zimbra Mail Routing Problem:
https://wiki.zimbra.com/wiki/Mail_Routing_Issues

...........................

# Verify zimbra status
su zimbra
zmcontrol status

# Restart zimbra Services
su zimbra
zmcontrol restart

---

এই ত্রুটিটি (**"system failure: exception during auth {RemoteManager: mail.pedrollobd.com->zimbra@mail.pedrollobd.com:22}"**) Zimbra মেইল সার্ভারে **SSH-ভিত্তিক রিমোট ম্যানেজমেন্ট** সংক্রান্ত একটি সমস্যা নির্দেশ করছে।

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



