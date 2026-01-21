# 📧 Zimbra Mail Server on Docker – সম্পূর্ণ গাইড (Beginner-Friendly)

> **লক্ষ্য**: এই গাইডটি আপনাকে শূন্য থেকে **Docker** এবং **docker-compose** ব্যবহার করে **Zimbra 8.8.15** মেইল সার্ভার চালু করতে সাহায্য করবে।  
> **অডিয়েন্স**: নন-টেকনিক্যাল বা বেসিক লিনাক্স জ্ঞান থাকলেই চলবে।  
> **প্ল্যাটফর্ম**: CentOS 7/8, Rocky Linux 8/9 বা Ubuntu 20.04+ (Host OS)  
> **Zimbra Version**: 8.8.15 GA (Community Edition)

---

## 📌 গুরুত্বপূর্ণ তথ্য

- ✅ **Zimbra অফিশিয়ালি Docker সাপোর্ট করে না**, কিন্তু আমরা systemd + privileged container ব্যবহার করে চালাব।
- ⚠️ **এটি শুধুমাত্র ল্যাব / শিক্ষার জন্য** — Production-এ VM বা Bare Metal ব্যবহার করুন।
- 🌐 আপনার ডোমেইন: `paulco.xyz` (আপনার নিজের ডোমেইন দিয়ে প্রতিস্থাপন করুন)
- 🖥️ হোস্ট মেশিনে **Docker** এবং **docker-compose** ইনস্টল থাকতে হবে।

---

## 🧰 ধাপ 1: হোস্ট মেশিন প্রস্তুতি

### 1.1 Docker এবং docker-compose ইনস্টল করুন

```bash
# Docker install (Ubuntu/CentOS/Rocky সবক্ষেত্রে কাজ করে)
curl -fsSL https://get.docker.com | sh

# docker-compose install
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# গ্রুপ অ্যাড করুন (sudo ছাড়া docker চালানোর জন্য)
sudo usermod -aG docker $USER
newgrp docker  # অথবা লগআউট/লগইন করুন
```

### 1.2 systemd-resolved বন্ধ করুন (যদি চালু থাকে)

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm -f /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

> 💡 কারণ: Zimbra নিজের DNS ব্যবহার করবে, conflict এড়াতে এটি জরুরি।

---

## 📁 ধাপ 2: প্রজেক্ট ফোল্ডার তৈরি করুন

```bash
mkdir -p ~/zimbra-docker/{config,data,logs}
cd ~/zimbra-docker
```

---

## 🌡️ ধাপ 3: `.env` ফাইল তৈরি করুন (কনফিগারেশন সেন্ট্রালাইজ)

```bash
cat > .env << 'EOF'
# Zimbra Mail Server Configuration
DOMAIN=paulco.xyz
HOSTNAME=mail.paulco.xyz
CONTAINER_IP=172.20.0.10
NETWORK_NAME=zimbra-net
TIMEZONE=Asia/Dhaka
EOF
```

> 🔁 **আপনার ডোমেইন দিয়ে `paulco.xyz` প্রতিস্থাপন করুন**  
> 🕒 `TIMEZONE` আপনার লোকেশন অনুযায়ী পরিবর্তন করুন: [List of Timezones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

---

## 🌐 ধাপ 4: কাস্টম ডকার নেটওয়ার্ক তৈরি (Static IP এর জন্য)

```bash
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/24 \
  --gateway=172.20.0.1 \
  zimbra-net
```

> 📌 এটি নিশ্চিত করে যে কন্টেইনারের IP সবসময় `172.20.0.10` থাকবে (`.env` এ সেট করা)।

---

## 📄 ধাপ 5: `docker-compose.yml` তৈরি করুন

```yaml
# docker-compose.yml
version: '3.8'

services:
  zimbra:
    image: rockylinux:8
    container_name: zimbra-mail
    hostname: ${HOSTNAME}
    domainname: ${DOMAIN}
    privileged: true
    restart: unless-stopped
    networks:
      zimbra-net:
        ipv4_address: ${CONTAINER_IP}
    environment:
      - TZ=${TIMEZONE}
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
      - ./data:/opt/zimbra
      - ./logs:/var/log/zimbra
      - ./config/dnsmasq.conf:/etc/dnsmasq.conf:ro
      - ./config/hosts:/etc/hosts:ro
    ports:
      - "25:25"
      - "53:53/udp"
      - "80:80"
      - "443:443"
      - "465:465"
      - "587:587"
      - "993:993"
      - "995:995"
      - "7071:7071"
      - "8080:8080"
      - "8443:8443"
      - "7110:7110"
      - "7143:7143"
    command: ["/sbin/init"]

networks:
  zimbra-net:
    external: true
```

> ✅ এখানে আমরা:
> - `privileged: true` → systemd চালানোর জন্য
> - `volumes` → ডেটা পার্সিস্টেন্স ও কনফিগ ম্যাপিং
> - `static IP` → DNS স্থিতিশীল রাখার জন্য

---

## 🛠️ ধাপ 6: DNS কনফিগারেশন ফাইল তৈরি করুন

### 6.1 `./config/hosts`

```bash
cat > config/hosts << EOF
127.0.0.1   localhost
${CONTAINER_IP} ${HOSTNAME} mail
EOF
```

### 6.2 `./config/dnsmasq.conf`

```bash
cat > config/dnsmasq.conf << EOF
server=8.8.8.8
domain=${DOMAIN}
mx-host=${DOMAIN},${HOSTNAME},10
listen-address=127.0.0.1
listen-address=${CONTAINER_IP}
address=/${HOSTNAME}/${CONTAINER_IP}
address=/${DOMAIN}/${CONTAINER_IP}
EOF
```

> 💡 এই ফাইলগুলো কন্টেইনারের ভিতরে DNS resolve করবে।

---

## ▶️ ধাপ 7: কন্টেইনার চালু করুন

```bash
cd ~/zimbra-docker
docker-compose up -d
```

> ⏳ প্রথমবারে কন্টেইনার শুধু চালু হবে — Zimbra install হবে না।

---

## 🔧 ধাপ 8: Zimbra ইনস্টল করুন (কন্টেইনারের ভিতরে)

### 8.1 কন্টেইনারে লগইন

```bash
docker exec -it zimbra-mail bash
```

### 8.2 YUM Repo Fix (CentOS End-of-Life)

```bash
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum clean all && yum makecache
```

### 8.3 প্রয়োজনীয় প্যাকেজ ইনস্টল

```bash
yum update -y
yum install -y nano wget tar unzip net-tools sysstat openssh-clients perl-core libaio nmap-ncat libstdc++ bind-utils dnsmasq
```

### 8.4 SELinux বন্ধ করুন

```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

### 8.5 dnsmasq চালু করুন

```bash
systemctl start dnsmasq
systemctl enable dnsmasq
```

### 8.6 DNS টেস্ট করুন

```bash
dig MX ${DOMAIN}
host -t A ${HOSTNAME}
# উভয়ের আউটপুটে আপনার CONTAINER_IP দেখাতে হবে
```

### 8.7 Zimbra ডাউনলোড ও ইনস্টল

```bash
cd /root
wget https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_4362.RHEL8_64.20220721104405.tgz
tar xvzf zcs-*.tgz
mv zcs-* zcs
cd zcs
./install.sh
```

> 📝 **ইনস্টলেশন সময়**:
> - Hostname: `mail.paulco.xyz` (auto)
> - DNS MX check: **Yes** (dnsmasq কাজ করছে কিনা চেক করবে)
> - Admin password: একটি strong password দিন
> - বাকি সব ডিফল্ট রাখুন

---

## 🎯 ধাপ 9: Zimbra অ্যাক্সেস করুন

- **Webmail**: `http://mail.paulco.xyz` বা `https://<your-server-ip>`
- **Admin Console**: `https://mail.paulco.xyz:7071`

> 🔐 প্রথমবারে HTTP দিয়ে ঢুকলে অটো redirect HTTPS-এ হবে।

---

## 💾 ধাপ 10: ডেটা ব্যাকআপ ও পুনরায় ব্যবহার

আপনার মেইল ডেটা `~/zimbra-docker/data` ফোল্ডারে সংরক্ষিত।  
কন্টেইনার ডিলিট করলেও ডেটা থাকবে।

পুনরায় চালাতে:

```bash
cd ~/zimbra-docker
docker-compose down
docker-compose up -d
# তারপর কন্টেইনারে লগইন করে Zimbra services start করুন (যদি অটো না হয়)
```

> 💡 Zimbra সার্ভিস অটো স্টার্ট হওয়া উচিত। না হলে:
> ```bash
> su - zimbra
> zmcontrol start
> ```

---

## ❓ সাধারণ সমস্যা ও সমাধান

| সমস্যা | সমাধান |
|--------|--------|
| **DNS MX check failed** | `dnsmasq` চালু আছে কিনা চেক করুন; `/etc/resolv.conf`-এ `nameserver 127.0.0.1` আছে কিনা |
| **Port already in use** | হোস্টে Postfix/Sendmail বন্ধ করুন: `sudo systemctl stop postfix` |
| **Zimbra services not starting** | `su - zimbra` → `zmcontrol status` → `zmcontrol start` |
| **Web UI not loading** | ফায়ারওয়াল চেক করুন; `sudo ufw allow 80,443/tcp` |

---

## 📦 বোনাস: কাস্টম Zimbra Image তৈরি (Optional)

ইনস্টল শেষে কমিট করুন:

```bash
docker commit zimbra-mail my-zimbra:8.8.15
```

তারপর `docker-compose.yml`-এ `image: my-zimbra:8.8.15` ব্যবহার করুন।

---

## 🌟 সমাপ্তি

> **“যে মেইল সার্ভার নিজে বানায়, সে কখনো মেইল হারায় না।”**  
> এই গাইড আপনাকে শুধু Zimbra চালানো শেখায় না — এটি আপনাকে **DNS, networking, systemd, এবং containerization**-এর গভীরে নিয়ে যায়।

✅ এখন আপনার নিজস্ব মেইল সার্ভার তৈরি হয়ে গেছে!  
📧 `user@paulco.xyz` ঠিকানা দিয়ে মেইল পাঠান ও গ্রহণ করুন।

---

> ℹ️ **Note**: এই সেটআপ **localhost বা লোকাল নেটওয়ার্কে** কাজ করবে।  
> ইন্টারনেট থেকে অ্যাক্সেস করতে চাইলে:
> - ডোমেইনের DNS-এ **A record** (`mail.paulco.xyz → your-public-ip`)
> - **MX record** (`paulco.xyz → mail.paulco.xyz`)
> - রাউটারে **port forwarding** (25, 80, 443, 587, 993, 995, 7071)
> - **SSL Certificate** (Let’s Encrypt)

---

**Happy Mailing!** 🚀  
*— আপনার DevOps সঙ্গী*