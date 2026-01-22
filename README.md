# 📧 Zimbra Mail Server: Comprehensive Deployment & Administration Guide

> **A professional, production-ready reference for DevOps engineers, system administrators, and IT teams managing enterprise email infrastructure.**

---

## 🎯 Objective

This repository provides a **complete, structured, and battle-tested knowledge base** for deploying, securing, troubleshooting, and operating **Zimbra Collaboration Suite** in real-world environments — from single-server setups to ISP-grade deployments with Barracuda gateways, Docker containers, and advanced security policies.

Whether you're:
- Setting up your first Zimbra server
- Managing a compromised account
- Automating deployment with Docker or scripts
- Integrating with Jitsi Meet or external SMTP relays

— this guide gives you **step-by-step, copy-paste ready instructions** backed by field experience.

---

## 📋 Table of Contents

1. [📌 What is Zimbra?](#-what-is-zimbra)
2. [🔍 Why Use Zimbra?](#-why-use-zimbra)
3. [📅 When to Use Zimbra](#-when-to-use-zimbra)
4. [✨ Key Features](#-key-features)
5. [✅ Pros & ❌ Cons](#-pros--cons)
6. [🆚 Comparison with Other Mail Servers](#-comparison-with-other-mail-servers)
7. [⚙️ Installation Methods](#️-installation-methods)
8. [🔧 Post-Installation Tasks](#-post-installation-tasks)
9. [💻 Practical CLI & Web Management Guide](#-practical-cli--web-management-guide)
10. [🛡️ Security Hardening & Best Practices](#️-security-hardening--best-practices)
11. [🧩 Advanced Topics & Integrations](#-advanced-topics--integrations)
12. [📂 Repository Structure & Related Documents](#-repository-structure--related-documents)
13. [📚 References & Further Reading](#-references--further-reading)

---

## 📌 What is Zimbra?

**Zimbra Collaboration Suite (ZCS)** is an open-source (with commercial edition) email and collaboration platform that provides:

- Full-featured webmail (modern & classic UI)
- Calendar, contacts, tasks, and document sharing
- Admin console for user/domain management
- Built-in antivirus (ClamAV), anti-spam (SpamAssassin), and MTA (Postfix)
- REST API, SOAP API, and CLI tools (`zmprov`, `zmmailbox`, etc.)
- Support for IMAP, POP3, ActiveSync, CalDAV, CardDAV

It’s used by businesses, universities, and ISPs worldwide as a **Microsoft Exchange alternative**.

---

## 🔍 Why Use Zimbra?

- ✅ **All-in-one**: Email + calendar + chat + file sharing
- ✅ **Self-hosted**: Full control over data and compliance
- ✅ **Open Source Core**: Free version available with enterprise features
- ✅ **Scalable**: From 10 users to 100,000+
- ✅ **Extensible**: Plugins for Jitsi, 2FA, SSO, etc.
- ✅ **CLI-first**: Ideal for automation and DevOps workflows

---

## 📅 When to Use Zimbra

| Scenario | Recommendation |
|--------|----------------|
| Small business needing private email | ✅ Ideal |
| ISP offering hosted email | ✅ Widely used |
| Enterprise replacing Exchange | ✅ Strong alternative |
| Personal email | ⚠️ Overkill — use simpler solutions |
| Cloud-native microservices | ❌ Not container-native (use Mailu, Mailcow instead) |

> 💡 **Note**: Zimbra is **not designed for Kubernetes** or ephemeral containers. It assumes persistent, stateful VMs.

---

## ✨ Key Features

| Category | Features |
|--------|--------|
| **Email** | Webmail, IMAP/POP3, SMTP, aliases, distribution lists |
| **Collaboration** | Shared calendars, contacts, briefcase (file storage) |
| **Security** | SPF, DKIM, DMARC, TLS, antivirus, spam filtering |
| **Admin Tools** | Web UI + powerful CLI (`zmprov`, `zmcontrol`) |
| **Mobile** | ActiveSync support (iOS/Android) |
| **Integrations** | Jitsi Meet, Outlook (via ZCO), LDAP, SAML |

---

## ✅ Pros & ❌ Cons

### ✅ Advantages
- Mature, stable, and feature-rich
- Excellent web interface
- Strong CLI for automation
- Good documentation and community
- Supports large-scale deployments

### ❌ Limitations
- Heavy resource usage (min 4GB RAM, 2 vCPU)
- Complex setup (DNS, hostname, SELinux critical)
- Not cloud-native (stateful, systemd-dependent)
- Open Source Edition lacks 2FA, SSO, and some admin tools
- Upgrades can be disruptive

---

## 🆚 Comparison with Other Mail Servers

| Feature | Zimbra | Postfix + Dovecot | Microsoft Exchange | Mailcow |
|-------|--------|------------------|-------------------|--------|
| Webmail | ✅ Built-in | ❌ (Roundcube needed) | ✅ Outlook Web | ✅ (Rainloop/SOGo) |
| Calendar | ✅ | ❌ | ✅ | ✅ |
| CLI Automation | ✅ Excellent | ⚠️ Manual config | ❌ PowerShell only | ⚠️ Limited |
| Container Support | ❌ (Not recommended) | ✅ | ❌ | ✅ |
| Resource Usage | High | Low | Very High | Medium |
| Learning Curve | Steep | Moderate | Steep | Moderate |

> 📌 **Verdict**: Choose Zimbra if you need **enterprise collaboration** on-premises. Choose Mailcow/Postfix if you want **lightweight, containerized email**.

---

## ⚙️ Installation Methods

Zimbra can be installed via several approaches:

### 1. **Bare Metal / VM (Recommended)**
- Install on CentOS/Rocky Linux or Ubuntu
- Requires proper DNS, hostname, and firewall setup
- Full control and best performance

📁 See:  
- [`/docs/installation/Install Zimbra Mail & DNS Server on CentOS.txt`](./docs/installation/Install%20Zimbra%20Mail%20%26%20DNS%20Server%20on%20CentOS.txt)  
- [`/docs/installation/Install Zimbra Mail & dnsmasq on Ubuntu.txt`](./docs/installation/Install%20Zimbra%20Mail%20%26%20dnsmasq%20on%20Ubuntu.txt)

### 2. **Docker (Lab/Dev Only)**
- Possible using `--privileged` + systemd
- Not supported in production
- Useful for testing or demos

📁 See:  
- [`/docs/installation/docker/zimbra-on-docker.md`](./docs/installation/docker/zimbra-on-docker.md)  
- [`/docs/installation/docker/zimbra-docker-compose.md`](./docs/installation/docker/zimbra-docker-compose.md)  
- [`/docs/installation/docker/zimbra-dockercompose-automation.md`](./docs/installation/docker/zimbra-dockercompose-automation.md)

### 3. **Automated Scripts**
- Silent install using response files
- Ideal for CI/CD or repeatable deployments

📁 See:  
- [`/docs/installation/docker/zimbra-dockercompose-automation.md`](./docs/installation/docker/zimbra-dockercompose-automation.md)

> ⚠️ **Warning**: Zimbra **officially recommends bare metal or VM**. Docker is for development only.

---

## 🔧 Post-Installation Tasks

After installing Zimbra, perform these critical steps:

### 1. **Configure DNS Records**
- **A Record**: `mail.yourdomain.com → Public IP`
- **MX Record**: `yourdomain.com → mail.yourdomain.com`
- **SPF**: `"v=spf1 mx a ip4:YOUR_IP -all"`
- **DKIM**: Generate via `zmdkimkeyutil`, add to DNS
- **DMARC**: `_dmarc.yourdomain.com IN TXT "v=DMARC1; p=quarantine; rua=mailto:admin@..."`

📁 See:  
- [`/docs/installation/Zimbra Mail  SPF, DKIM and DMARC Setup.txt`](./docs/installation/Zimbra%20Mail%20%20SPF,%20DKIM%20and%20DMARC%20Setup.txt)

### 2. **Secure the Server**
- Disable SELinux (`setenforce 0`, edit `/etc/selinux/config`)
- Stop conflicting services (`postfix`, `sendmail`)
- Configure firewall (`firewalld`/`ufw`)

### 3. **Create Users & Domains**
```bash
su - zimbra
zmprov cd yourdomain.com
zmprov ca user@yourdomain.com 'StrongPass123!'
```

### 4. **Enable Services**
- Start all services: `zmcontrol start`
- Check status: `zmcontrol status`

---

## 💻 Practical CLI & Web Management Guide

### 🔑 Essential CLI Commands

| Task | Command |
|------|--------|
| Create user | `zmprov ca user@domain.com 'Pass'` |
| Change password | `zmprov sp user@domain.com 'NewPass'` |
| List users | `zmaccts` |
| View mail queue | `mailq` or `/opt/zimbra/common/sbin/postqueue -p` |
| Delete queued mail | `/opt/zimbra/common/sbin/postsuper -d QUEUE_ID` |
| Restart services | `zmcontrol restart` |
| Check mailbox size | `zmprov gmi user@domain.com` |

📁 Full CLI Guide:  
- [`/zimbra-cli-manage-guide-eng.md`](./zimbra-cli-manage-guide-eng.md)  
- [`/zimbra-cli-manage-guide-bng.md`](./zimbra-cli-manage-guide-bng.md)

### 🖥️ Web Admin Console
- URL: `https://mail.yourdomain.com:7071`
- Manage users, domains, COS, filters, and logs
- Enable features like chat, drive, and mobile sync

---

## 🛡️ Security Hardening & Best Practices

### 🔒 Critical Actions
1. **Enforce strong passwords** (via Class of Service)
2. **Set up SPF/DKIM/DMARC** — prevents spoofing
3. **Whitelist trusted networks**:  
   ```bash
   zmprov ms $(zmhostname) zimbraMtaMyNetworks '127.0.0.0/8 192.168.0.0/24'
   ```
4. **Block spam senders**:  
   Edit `/opt/zimbra/conf/salocal.cf.in` or use Amavis rules

📁 See:  
- [`/Whitelist-Blacklist Domain in Zimbra .md`](./Whitelist-Blacklist%20Domain%20in%20Zimbra%20.md)  
- [`/docs/error&solutions/compromised-detection-prevention`](./docs/error&solutions/compromised-detection-prevention)

### 🚨 Handle Compromised Accounts
- Detect via: `grep 'sasl_username=' /opt/zimbra/log/mailbox.log`
- Delete spam queue safely
- Reset password, enable 2FA, suspend account

📁 See:  
- [`/docs/error&solutions/compromise-internal-mails.md`](./docs/error&solutions/compromise-internal-mails.md)

---

## 🧩 Advanced Topics & Integrations

### 🔗 Integrate with Jitsi Meet
- Use LDAP authentication
- Install Zimlet for one-click video calls

📁 See:  
- [`/docs/Integrate Zimbra mail with jitsi meet.txt`](./docs/Integrate%20Zimbra%20mail%20with%20jitsi%20meet.txt)

### 📤 Outgoing SMTP Relay (e.g., Barracuda)
- Configure Zimbra to relay via external gateway
- Monitor logs on both Zimbra and Barracuda

📁 See:  
- [`/docs/zimbra-outgoing-smtp-logs.md`](./docs/zimbra-outgoing-smtp-logs.md)

### 🐳 Run in Docker (Advanced)
- Use custom network, static IP, and volume persistence
- Automate with `docker-compose` + entrypoint script

📁 See:  
- [`/docs/installation/docker/`](./docs/installation/docker/)

---

## 📚 References & Further Reading

- [Zimbra Official Wiki](https://wiki.zimbra.com/)
- [Zimbra Forums](https://forums.zimbra.org/)
- [MXToolbox – Email Health Checker](https://mxtoolbox.com/)
- [DKIM Validator](https://dkimcore.org/tools/)
- [SPF Record Generator](https://mxtoolbox.com/spf.aspx)
- [Zimbra to Zimbra Migration](https://wiki.zimbra.com/wiki/Zimbra_to_Zimbra_Migration)
- [How to use the new Zimbra Migration Tool - pst file to Zimbra Desktop](https://blog.zimbra.com/2017/01/use-new-zimbra-migration-tool-pst-file-zimbra-desktop/)
- [Running Migration Wizard](https://wiki.zimbra.com/wiki/Running_Migration_Wizard)

---

## 🙏 Final Notes

> **“Email is the backbone of business communication. A well-configured Zimbra server isn’t just a mail system — it’s a trust anchor.”**

This guide is maintained by **Sumon Paul**, a DevOps engineer with 8+ years of experience in IT infrastructure, cloud, and email systems. It reflects real-world deployments in Bangladesh and global environments.

Use it wisely. Secure your data. And never skip SPF/DKIM!

---

**✅ Prepared for**: System Administrators, DevOps Engineers, ISPs  
**🌐 Tested on**: Zimbra 8.8.x, 9.0, 10.x (Open Source & Network Edition)  
**📅 Last Updated**: January 2026  
**📍 Location**: Dhaka, Bangladesh  

---

> 📩 **Need help?** This repo includes ready-to-use scripts, configs, and troubleshooting flows.  
> 🔒 **Remember**: Always backup (`zmbackup`) before major changes!

---

