# 🚀 **স্বয়ংক্রিয়ভাবে Zimbra Mail Server on Docker – সম্পূর্ণ অটোমেটেড গাইড**

> ✅ **লক্ষ্য**: Zimbra কন্টেইনার চালু হওয়ার সাথে সাথে **সম্পূর্ণ অটোমেটিকভাবে** Zimbra 8.8.15 ইনস্টল, কনফিগার ও চালু হবে — **কন্টেইনারে ম্যানুয়ালি ঢোকার প্রয়োজন নেই**।  
> 🔧 **পদ্ধতি**: `Dockerfile` + `entrypoint.sh` + `response file` + `docker-compose`  
> ⚠️ **উদ্দেশ্য**: শুধুমাত্র **ল্যাব / ডেভেলপমেন্ট / ডেমো** পরিবেশের জন্য। Production-এ VM ব্যবহার করুন।

---

## 📌 কেন এই পদ্ধতি?

- ❌ ম্যানুয়াল ইনস্টল → সময়সাপেক্ষ, ভুলের সম্ভাবনা
- ✅ **অটোমেটেড install.sh** → Zimbra-র `-s` (silent) মোড ব্যবহার করে
- ✅ **একবার সেটআপ = যেকোনো মেশিনে রিপ্লিকেট**
- ✅ **docker-compose + .env** → কনফিগারেশন সেন্ট্রালাইজড

---

## 🗂️ প্রজেক্ট স্ট্রাকচার

```bash
zimbra-auto/
├── .env
├── docker-compose.yml
├── Dockerfile
├── entrypoint.sh
├── zimbra-response.txt
└── config/
    ├── dnsmasq.conf
    └── hosts
```

---

## 📄 ধাপ 1: `.env` — কনফিগারেশন ভেরিয়েবল

```env
# .env
DOMAIN=paulco.xyz
HOSTNAME=mail.paulco.xyz
ADMIN_PASSWORD=ZimbraAdmin@123
CONTAINER_IP=172.20.0.10
NETWORK_NAME=zimbra-net
TIMEZONE=Asia/Dhaka
```

> 🔐 **ADMIN_PASSWORD** অবশ্যই strong রাখুন।

---

## 🌐 ধাপ 2: কাস্টম ডকার নেটওয়ার্ক তৈরি

```bash
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/24 \
  --gateway=172.20.0.1 \
  zimbra-net
```

---

## 📝 ধাপ 3: `zimbra-response.txt` — Silent Install Config

```txt
# zimbra-response.txt
HOSTNAME=${HOSTNAME}
EMAIL=admin@${DOMAIN}
PASSWORD=${ADMIN_PASSWORD}
SMTPNOTIFY=no
CREATEADMIN=yes
ADMINLOGIN=admin@${DOMAIN}
ADMINPASSWORD=${ADMIN_PASSWORD}
SLAVELDAP=no
SLAVELDAPURL=
INSTALL_PACKAGES="zimbra-core zimbra-ldap zimbra-logger zimbra-mta zimbra-snmp zimbra-store zimbra-apache zimbra-spell zimbra-proxy"
MODE=http
ENABLEIMAP=yes
ENABLEIMAPS=yes
ENABLEPOP3=yes
ENABLEPOP3S=yes
DISABLEUPDATECHECK=yes
```

> 💡 এই ফাইলটি Zimbra-র silent install-এর জন্য প্রয়োজন।

---

## 🐚 ধাপ 4: `entrypoint.sh` — অটোমেশন স্ক্রিপ্ট

```bash
#!/bin/bash
set -e

echo "🔧 [1/5] Setting hostname..."
hostnamectl set-hostname ${HOSTNAME}

echo "🔧 [2/5] Fixing YUM repos (CentOS EOL)..."
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum clean all && yum makecache

echo "🔧 [3/5] Installing dependencies..."
yum install -y wget tar unzip net-tools sysstat perl-core libaio nmap-ncat libstdc++ bind-utils dnsmasq

echo "🔧 [4/5] Disabling SELinux..."
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0

echo "🔧 [5/5] Starting dnsmasq..."
systemctl start dnsmasq

# Wait for DNS to be ready
sleep 5

# Expand env vars in response file
envsubst < /tmp/zimbra-response.txt > /tmp/response-final.txt

# Download & Install Zimbra
if [ ! -f /opt/zimbra/.installed ]; then
  echo "📥 Downloading Zimbra..."
  cd /tmp
  wget -q https://files.zimbra.com/downloads/8.8.15_GA/zcs-8.8.15_GA_4362.RHEL8_64.20220721104405.tgz
  tar -xzf zcs-*.tgz
  mv zcs-* zcs

  echo "⚙️ Installing Zimbra silently..."
  cd zcs
  ./install.sh -s < /tmp/response-final.txt

  # Mark as installed
  touch /opt/zimbra/.installed
fi

echo "✅ Zimbra setup complete! Starting systemd..."
exec /sbin/init
```

> 📝 `envsubst` ব্যবহার করে `.env` ভেরিয়েবলগুলো `zimbra-response.txt`-এ inject করা হয়।

---

## 🐳 ধাপ 5: `Dockerfile`

```Dockerfile
# Dockerfile
FROM rockylinux:8

# Install envsubst (from gettext)
RUN yum install -y gettext && yum clean all

# Copy automation files
COPY zimbra-response.txt /tmp/
COPY entrypoint.sh /usr/local/bin/

# Make executable
RUN chmod +x /usr/local/bin/entrypoint.sh

# Expose Zimbra ports
EXPOSE 25 53/udp 80 443 465 587 993 995 7071 8080 8443 7110 7143

# Use systemd-compatible init
CMD ["/usr/local/bin/entrypoint.sh"]
```

---

## 📄 ধাপ 6: `docker-compose.yml`

```yaml
# docker-compose.yml
version: '3.8'

services:
  zimbra:
    build: .
    container_name: zimbra-mail
    hostname: ${HOSTNAME}
    domainname: ${DOMAIN}
    privileged: true
    restart: unless-stopped
    networks:
      zimbra-net:
        ipv4_address: ${CONTAINER_IP}
    environment:
      - DOMAIN=${DOMAIN}
      - HOSTNAME=${HOSTNAME}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - TZ=${TIMEZONE}
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
      - zimbra_data:/opt/zimbra
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

volumes:
  zimbra_data:

networks:
  zimbra-net:
    external: true
```

---

## 📁 ধাপ 7: DNS কনফিগ ফাইল

### `config/hosts`
```txt
127.0.0.1 localhost
172.20.0.10 mail.paulco.xyz mail
```

### `config/dnsmasq.conf`
```conf
server=8.8.8.8
domain=paulco.xyz
mx-host=paulco.xyz,mail.paulco.xyz,10
listen-address=127.0.0.1
listen-address=172.20.0.10
address=/mail.paulco.xyz/172.20.0.10
address=/paulco.xyz/172.20.0.10
```

> 🔄 **`.env`-এর মান অনুযায়ী IP/Domain আপডেট করুন**।

---

## ▶️ ধাপ 8: এক কমান্ডে চালু করুন!

```bash
# 1. প্রজেক্ট ফোল্ডারে যান
cd zimbra-auto

# 2. কম্পোজ বিল্ড ও চালু করুন
docker-compose up -d --build
```

> ⏳ **প্রথম রানে ~10-15 মিনিট সময় লাগবে** (Zimbra download + install)।  
> 📊 লগ দেখুন: `docker logs -f zimbra-mail`

---

## ✅ ধাপ 9: যাচাইকরণ

```bash
# Zimbra services running?
docker exec zimbra-mail su - zimbra -c "zmcontrol status"

# Web UI accessible?
curl -k https://localhost:7071  # should return HTML
```

- **Webmail**: `http://<your-server-ip>`
- **Admin Console**: `https://<your-server-ip>:7071`  
  → Username: `admin@paulco.xyz`  
  → Password: `ZimbraAdmin@123` (`.env` থেকে)

---

## 💡 অতিরিক্ত টিপস: Docker-এ Zimbra চালানোর জন্য

### 1. **Resource Allocation**
Zimbra কমপক্ষে **4 GB RAM**, **2 vCPU** চায়। Docker Desktop-এ resource limit বাড়ান।

### 2. **Volume Persistence**
`zimbra_data` volume ব্যবহার করে মেইল ডেটা সংরক্ষিত থাকবে।  
ব্যাকআপ: `docker run --rm -v zimbra_data:/data -v $(pwd):/backup alpine tar czf /backup/zimbra-backup.tar.gz -C /data .`

### 3. **SSL Certificate (Let’s Encrypt)**
Zimbra ডিফল্ট self-signed cert ব্যবহার করে। Production-এ:
```bash
# Zimbra-র ভিতরে
su - zimbra
/opt/zimbra/bin/zmcertmgr deploycrt comm /path/to/cert.pem /path/to/key.pem
```

### 4. **External Access**
- ডোমেইনে A রেকর্ড: `mail.paulco.xyz → your-public-ip`
- MX রেকর্ড: `paulco.xyz → mail.paulco.xyz`
- রাউটারে port forward: 25, 80, 443, 587, 993, 995, 7071

### 5. **Security Hardening**
- ফায়ারওয়ালে শুধু প্রয়োজনীয় পোর্ট open করুন
- Admin console (`7071`) শুধু ভিপিএন থেকে অ্যাক্সেস করুন
- নিয়মিত Zimbra আপডেট চেক করুন

---

## 🔄 আপগ্রেড / রিবিল্ড

```bash
# নতুন ভার্সনে আপগ্রেড করতে চাইলে:
docker-compose down
# Dockerfile-এ নতুন Zimbra URL দিন
docker-compose up -d --build
```

> ⚠️ **Data volume (`zimbra_data`) রিমুভ করবেন না** — মেইল ডেটা হারিয়ে যাবে!

---

## 🎯 সারসংক্ষেপ

| বৈশিষ্ট্য | বিবরণ |
|----------|--------|
| **অটোমেশন** | কন্টেইনার চালু → Zimbra অটো install |
| **কনফিগারেশন** | `.env` ফাইলে সবকিছু কেন্দ্রীভূত |
| **পুনরায় ব্যবহারযোগ্য** | যেকোনো মেশিনে `docker-compose up` |
| **ডেটা সুরক্ষা** | Docker volume ব্যবহার করে persistence |

---

> 🌟 **“Automation is not about replacing humans — it’s about freeing them to solve harder problems.”**  
> এই সেটআপ আপনাকে Zimbra-র জটিলতা থেকে মুক্তি দেবে, যাতে আপনি **DevOps, Security, বা Integration**-এ ফোকাস করতে পারেন।

**Happy Automating!** 🐳📧