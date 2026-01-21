নিচে আপনার দেওয়া Zimbra-on-Docker কনফিগারেশনগুলোকে **সুসংগঠিত, ধাপে ধাপে, প্রয়োজনীয় সংশোধনসহ** উপস্থাপন করা হলো। এটি **CentOS 7/8 বা Rocky Linux**-ভিত্তিক container-এ Zimbra 8.8.15 ইনস্টল করার জন্য কার্যকর গাইডলাইন হিসাবে কাজ করবে। Ubuntu ভার্সনটি আলাদাভাবে উল্লেখ করা হয়েছে, কিন্তু **Zimbra official support CentOS/RHEL বেসের উপর ভিত্তি করে**, তাই CentOS/Rocky রেকমেন্ডেড।

---

## ✅ **Part 0: প্রাথমিক প্রস্তুতি (Host-Level)**

Zimbra **DNS, hostname, SELinux, port conflicts**-এর উপর অত্যন্ত সেনসিটিভ।

```bash
# 1. Host-এর systemd-resolved বন্ধ করুন (যদি চালু থাকে)
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.resolve
sudo ln -sf /etc/resolv.resolve /etc/resolv.conf

# 2. Docker হোস্টে ফায়ারওয়াল চেক করুন — প্রয়োজনীয় পোর্টগুলো allow করুন (পরে বলা হয়েছে)
```

---

## ✅ **Part 1: Zimbra Container তৈরি (CentOS/Rocky Linux Base)**

> ⚠️ **গুরুত্বপূর্ণ**: Zimbra **/sbin/init** বা **systemd** ছাড়া কাজ করে না। সুতরাং, container-এ systemd সাপোর্ট থাকা আবশ্যিক। CentOS/Rocky **"init"** image ব্যবহার করুন।

```bash
# Zimbra Container চালু করুন
docker run -d \
  --name zimbra \
  --hostname mail.paulco.xyz \
  --privileged \
  --tmpfs /tmp \
  --tmpfs /run \
  --tmpfs /run/lock \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  --add-host=mail.paulco.xyz:172.17.0.2 \
  -p 25:25 \
  -p 53:53/udp \
  -p 80:80 \
  -p 443:443 \
  -p 465:465 \
  -p 587:587 \
  -p 993:993 \
  -p 995:995 \
  -p 7071:7071 \
  -p 8080:8080 \
  -p 8443:8443 \
  -p 7110:7110 \
  -p 7143:7143 \
  rockylinux:8   # or centos:7

# মন্তব্য: --privileged + cgroup mount systemd চালানোর জন্য প্রয়োজন
```

> 📌 **IP Note**: `172.17.0.2` Docker-এর default bridge IP হতে পারে, কিন্তু নির্ভরযোগ্যতার জন্য **static container IP** বা **custom docker network** ব্যবহার করুন।

---

## ✅ **Part 2: Container-এ লগইন করে প্রস্তুতি**

```bash
docker exec -it zimbra bash
```

### 2.1 YUM Repo Fix (CentOS End-of-Life এর কারণে)

```bash
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum clean all && yum makecache
```

### 2.2 Essential Packages Install

```bash
yum update -y
yum install -y nano wget tar unzip net-tools sysstat openssh-clients perl-core libaio nmap-ncat libstdc++ bind-utils
```

### 2.3 SELinux Disable

```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

> 📌 **SELinux must be disabled** — Zimbra officially does not support it.

---

## ✅ **Part 3: Internal DNS (dnsmasq) Setup – অপরিহার্য**

Zimbra স্থানীয় MX, A রেকর্ড ঠিকমতো resolve করতে পারলে তবেই ইনস্টল হবে।

### 3.1 dnsmasq install

```bash
yum install -y dnsmasq
```

### 3.2 `/etc/dnsmasq.conf` কনফিগার

```bash
cp /etc/dnsmasq.conf /etc/dnsmasq.conf.bac

cat > /etc/dnsmasq.conf << EOF
server=8.8.8.8
domain=paulco.xyz
mx-host=paulco.xyz,mail.paulco.xyz,10
listen-address=127.0.0.1
listen-address=172.17.0.2
address=/mail.paulco.xyz/172.17.0.2
address=/paulco.xyz/172.17.0.2
EOF
```

### 3.3 `/etc/hosts` update

```bash
echo "172.17.0.2 mail.paulco.xyz mail" >> /etc/hosts
```

### 3.4 `/etc/resolv.conf` override

```bash
echo "nameserver 127.0.0.1" > /etc/resolv.conf
# অপশনাল: chattr +i /etc/resolv.conf (immutable করে দিন)
```

### 3.5 dnsmasq start & test

```bash
systemctl start dnsmasq
systemctl enable dnsmasq

# Test
dig MX paulco.xyz
dig A mail.paulco.xyz
host -t mx paulco.xyz
```

> ✅ সবগুলো কমান্ডে সঠিক আউটপুট আসতে হবে।

---

## ✅ **Part 4: Zimbra Install**

### 4.1 Download Zimbra

```bash
cd /root
wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_4362.RHEL8_64.20220721104405.tgz
tar xvzf zcs-*.tgz
mv zcs-* zcs
```

> 🔗 **Official Download**: https://www.zimbra.com/downloads/

### 4.2 Install Zimbra

```bash
cd zcs
./install.sh
```

- **Hostname**: `mail.paulco.xyz` (auto-detected, verify)
- **DNS MX check**: Should pass (thanks to dnsmasq)
- **Admin password**: Set strong password
- **TimeZone**: Set correctly

> ⚠️ **Installation শেষে**, Zimbra services অটো স্টার্ট হবে।

---

## ✅ **Part 5: পোর্ট ম্যাপিং চেক & ফায়ারওয়াল**

Zimbra-র জন্য নিম্নলিখিত পোর্টগুলো **host-এ open** থাকতে হবে:

| Service     | Port  | Protocol |
|-------------|-------|----------|
| SMTP        | 25    | TCP      |
| SMTPS       | 465   | TCP      |
| Submission  | 587   | TCP      |
| HTTP        | 80    | TCP      |
| HTTPS       | 443   | TCP      |
| IMAPS       | 993   | TCP      |
| POP3S       | 995   | TCP      |
| Admin UI    | 7071  | TCP      |
| Proxy UI    | 8080 / 8443 | TCP |

> 🔒 **Host Firewall (firewalld/ufw)**: সবগুলো allow করুন।

```bash
# firewalld example
sudo firewall-cmd --permanent --add-port={25,80,443,465,587,993,995,7071,8080,8443}/tcp
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --reload
```

---

## ✅ **Part 6: Zimbra Docker Image Build (Optional – for Reusability)**

Dockerfile ব্যবহার করে **reproducible** Zimbra image তৈরি করা সম্ভব, কিন্তু **Zimbra install process interactive**, তাই **Dockerfile এককালীন ইনস্টলের জন্য উপযোগী নয়**।

### ✅ সুপারিশকৃত পদ্ধতি:

1. উপরের মতো container-এ Zimbra install করুন
2. Install শেষে container commit করুন:

```bash
docker stop zimbra
docker commit zimbra zimbra-mail:8.8.15
```

3. পরবর্তীতে চালানোর জন্য:

```bash
docker run -d \
  --name zimbra-prod \
  --hostname mail.paulco.xyz \
  --privileged \
  --tmpfs /tmp --tmpfs /run --tmpfs /run/lock \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  -p 25:25 -p 80:80 -p 443:443 ... \
  zimbra-mail:8.8.15 \
  /sbin/init
```

> ❌ **শুধু `CMD ./install.sh` দিয়ে Dockerfile তৈরি করলে Zimbra install হবে না**, কারণ install.sh interactive, non-automated।

---

## ✅ **Part 7: Ubuntu (Alternative – Not Recommended for Zimbra 8.8.15)**

Zimbra **Ubuntu** support আছে, কিন্তু:

- সঠিক package নাম্বার মিলাতে হবে (`UBUNTU20_64`)
- systemd ছাড়া চলবে না
- কনফিগারেশন পার্থক্য আছে

```bash
docker run -d --name zimbra --hostname mail.paulco.xyz --privileged ubuntu:20.04 /sbin/init
# ... similar steps, but use apt, not yum
```

> 📌 **সুপারিশ**: Production-এ **CentOS/Rocky Linux** ব্যবহার করুন।

---

## ✅ **সতর্কতা ও Best Practices**

| বিষয় | মন্তব্য |
|------|--------|
| **systemd সমর্থন** | `--privileged + /sbin/init` ছাড়া Zimbra চলবে না |
| **DNS** | External DNS না থাকলে **dnsmasq বাধ্যতামূলক** |
| **Hostname** | FQDN (`mail.paulco.xyz`) ঠিক থাকতে হবে — `hostname -f` চেক করুন |
| **SELinux** | **বন্ধ করুন** |
| **Port Conflict** | Host-এ অন্য মেইল সার্ভার (Postfix, Sendmail) থাকলে বন্ধ করুন |
| **Resource** | Zimbra ন্যূনতম **4 GB RAM**, 2 vCPU চায় |

---

## ✅ **Zimbra অ্যাক্সেস**

- **Webmail**: `https://mail.paulco.xyz`
- **Admin Console**: `https://mail.paulco.xyz:7071`

---

## 🌱 **সমাপ্তি মন্তব্য**

Zimbra কেবল মেইল সার্ভার নয় — একটি **সম্পূর্ণ কলাবোরেশন স্যুট**। ডকারে চালানো চ্যালেঞ্জিং হলেও, সঠিক DNS, systemd ও পোর্ট ম্যানেজমেন্টের মাধ্যমে এটি সম্ভব। কিন্তু মনে রাখবেন: **Zimbra officially recommends bare-metal or VM**, ডকার হলো **lab/development use only**।
