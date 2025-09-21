# 📧 Zimbra Mail Server: Simple CLI Guide

> ✅ For Zimbra 8.x / 9.x / 10.x  
> ✅ Run most commands as `zimbra` user  
> ✅ Tested, Safe, and Practical  
> ✅ Save as `README.md` or print for quick reference

---

## 📋 Table of Contents

1. [📌 Introduction](#-introduction)  
2. [🌐 Check Your Server Info](#-check-your-server-info)  
3. [👤 Manage User Accounts](#-manage-user-accounts)  
4. [📬 Manage Mail Queue](#-manage-mail-queue)  
5. [🗑️ Stop Spam & Hacked Accounts](#️-stop-spam--hacked-accounts)  
6. [📊 Check Mailbox Sizes](#-check-mailbox-sizes)  
7. [🔐 Restrict Admin Access (No View Mail)](#-restrict-admin-access-no-view-mail)  
8. [⚙️ Restart & Fix Problems](#️-restart--fix-problems)  
9. [📜 Useful Scripts](#-useful-scripts)  
10. [📌 Emergency Cheat Sheet](#-emergency-cheat-sheet)  
11. [📚 Learn More](#-learn-more)

---

## 📌 Introduction

Zimbra is a powerful email server. The web admin panel is nice, but the **command line (CLI) is faster, stronger, and works when the web UI is slow or broken**.

> ⚠️ WARNING:  
> - Always start with: `su - zimbra`  
> - `postsuper -d ALL` deletes ALL emails — use carefully!  
> - Test commands first if you’re not sure.

---

## 🌐 Check Your Server Info

### Check Public IP (for spam checks)

```bash
curl ipinfo.io/ip
```

### Check Disk Space (if Zimbra won’t start)

```bash
df -h
lsblk
```

> If `/opt/zimbra` is full → Zimbra will crash!

---

## 👤 Manage User Accounts

### ➕ Create New User

```bash
su - zimbra
zmprov ca user@yourdomain.com 'StrongP@ss123!'
```

> `ca` = Create Account. Use strong passwords!

---

### ➖ Delete User (⚠️ Permanent!)

```bash
zmprov da user@yourdomain.com
```

> Deletes everything — mailbox, settings, history.

---

### 🔐 Change Password

```bash
zmprov sp user@yourdomain.com 'NewP@ssw0rd!'
```

> `sp` = Set Password. Tell the user after changing.

---

### 🔒 Lock or Unlock Account

```bash
# Lock (stop login)
zmprov ma user@domain.com zimbraAccountStatus locked

# Unlock (allow login)
zmprov ma user@domain.com zimbraAccountStatus active
```

> Use `locked` for hacked/spam accounts.

---

### 📜 List All Users

```bash
zmaccts
```

> Shows: email, status (active/locked), creation date.

---

### 👑 Make User an Admin

```bash
zmprov ma user@domain.com zimbraIsAdminAccount TRUE
```

---

### 👤 Remove Admin Rights

```bash
zmprov ma user@domain.com zimbraIsAdminAccount FALSE
```

---

## 📬 Manage Mail Queue

> 💡 Run as `zimbra` user unless noted.

### 🔍 View Mail Queue

```bash
mailq
# or
postqueue -p
```

> Shows: ID, Size, Sender, Recipient, Status.

---

### 🔄 Retry All Emails (after fixing network/DNS)

```bash
postqueue -f
```

---

### 🗑️ Delete One Email

```bash
postsuper -d ABC123DEF456
```

> Replace `ABC123DEF456` with real Queue ID from `mailq`.

---

### 🗑️ Delete ALL Emails (⚠️ Emergency Only!)

```bash
postsuper -d ALL
```

---

### 🗑️ Delete Emails for ONE Person (Safe Version)

```bash
/opt/zimbra/common/sbin/mailq | \
awk '/user@domain\.com/ {gsub(/^[*!]+/, "", $1); print $1}' | \
grep '^[0-9A-Fa-f]' | \
xargs -r /opt/zimbra/common/sbin/postsuper -d
```

> ✅ Removes `*!`, checks valid ID, deletes safely.

---

### 🗑️ Delete Only Deferred (Stuck) Emails

```bash
/opt/zimbra/common/sbin/postsuper -d ALL deferred
```

> Great for clearing spam backlog.

---

## 🗑️ Stop Spam & Hacked Accounts

### 🔍 Find Who is Sending Spam

```bash
cat /var/log/zimbra.log | sed -n 's/.*sasl_username=//p' | sort | uniq -c | sort -n
```

> Output: `   500 hacker@domain.com` ← This user is hacked!

---

### 🔐 IMMEDIATELY Change Password

```bash
zmprov sp hacker@domain.com 'BrandNewP@ss2024!'
```

> Do this BEFORE clearing queue!

---

### 🗑️ Delete All Their Emails from Queue

```bash
mailq | awk '/hacker@domain\.com/ {print $1}' | xargs -n1 postsuper -d
```

> `xargs -n1` = delete one by one → safer.

---

## 📊 Check Mailbox Sizes

### 📏 Check One User’s Size

```bash
zmprov gmi user@domain.com
```

> Shows size, item count, last login.

---

### 📏 Show Size in MB (Easy to Read)

```bash
zmmailbox -z -m user@domain.com gms | awk '{print $3/1024/1024 " MB"}'
```

---

### 📂 Show Folder Sizes (Inbox, Sent, etc.)

```bash
zmmailbox -z -m user@domain.com gaf
```

---

### 📊 Generate Report: All Users’ Mailbox Sizes

> ✅ Save as: `allmailboxsize.sh`

```bash
#!/bin/bash
echo "=== Zimbra Mailbox Size Report ==="
echo "Date: $(date)"
echo "=================================="

for account in $(zmprov -l gaa 2>/dev/null); do
    size=$(zmmailbox -z -m "$account" gms 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "$account = $size"
    else
        echo "ERROR: Cannot read $account"
    fi
done
```

> 🔧 Run it:

```bash
chmod +x allmailboxsize.sh
./allmailboxsize.sh > report_$(date +%Y%m%d).txt
```

> Saves to a file like: `report_20250405.txt`

---

## 🔐 Restrict Admin Access (No View Mail)

> Default Zimbra admins can read ALL user emails. You can block that!

### Step 1: Create Admin Group

```bash
zmprov cdl restricted-admins@yourdomain.com zimbraIsAdminGroup TRUE
```

### Step 2: Set What They Can See in Web UI

```bash
zmprov mdl restricted-admins@yourdomain.com \
zimbraAdminConsoleUIComponents accountListView \
zimbraAdminConsoleUIComponents DLListView \
zimbraAdminConsoleUIComponents domainListView \
zimbraAdminConsoleUIComponents serverListView \
zimbraAdminConsoleUIComponents globalConfigView
```

### Step 3: Give Admin Rights (Except “View Mail”)

```bash
zmprov grr global grp restricted-admins@yourdomain.com +domainAdminRights
zmprov grr global grp restricted-admins@yourdomain.com +adminConsoleAccountRights
zmprov grr global grp restricted-admins@yourdomain.com +adminConsoleServerRights
# ... add more rights if needed — but NEVER add "viewMail" or "adminLoginAs"
```

### Step 4: Apply to Your Domain

```bash
zmprov grr domain yourdomain.com grp restricted-admins@yourdomain.com +domainAdminRights
zmprov grr domain yourdomain.com grp restricted-admins@yourdomain.com -adminLoginAs
```

### Step 5: Create New Restricted Admin

```bash
zmprov ca newadmin@yourdomain.com 'Pass123!' zimbraIsDelegatedAdminAccount TRUE
zmprov adlm restricted-admins@yourdomain.com newadmin@yourdomain.com
```

### Step 6: Clear Cache

```bash
zmprov fc all
```

> ✅ Now `newadmin@yourdomain.com` CANNOT view user emails.

> 📘 Source: [Zimbra Wiki - Restrict View Mail](https://wiki.zimbra.com/wiki/Restrict_Admin_%27View_Mail%27)

---

## ⚙️ Restart & Fix Problems

### 📊 Check Service Status

```bash
zmcontrol status
```

> Shows: mailbox, mta, ldap, logger — if any say “not running”, fix it.

---

### 🔄 Restart All Services

```bash
zmcontrol restart
```

> Use after config changes or if server is slow.

---

### 🛠️ Fix File Permissions (After Crash)

```bash
/opt/zimbra/libexec/zmfixperms -e -v
```

> `-e` = fix more things, `-v` = show what it’s doing.

---

### 💾 Disk Full? Zimbra Won’t Start?

```bash
su - zimbra
df -h
# If /opt/zimbra is 100% full:
zmcontrol stop
# As ROOT, clean logs:
rm -f /opt/zimbra/log/*.log.*
# Then start again:
zmcontrol start
```

> Tip: Set up `logrotate` to auto-clean logs.

---

## 📜 Useful Scripts

### 📩 Delete Emails from User’s Inbox (by Date)

```bash
# List emails in Inbox from Jan 1, 2024
zmmailbox -z -m user@domain.com s -t message -l 50 "in:inbox date:01/01/2024"

# Delete email with ID 431
zmmailbox -z -m user@domain.com deleteMessage 431
```

> Great for removing phishing emails from many accounts.

---

### 📜 Monitor Admin Actions (Real-Time)

```bash
tail -f /opt/zimbra/log/audit.log | grep "admin@domain.com"
```

> See what admins are doing — login, delete, change settings.

---

## 📌 Emergency Cheat Sheet

| Problem | Quick Fix |
|---------|-----------|
| Server frozen | `zmcontrol restart` |
| Disk full | `df -h` → delete old logs → `zmcontrol start` |
| Spam flood | Lock account → Change password → Clear queue |
| Queue jammed | `postsuper -d ALL deferred` → `postqueue -f` |
| Web UI broken | Use CLI — `mailq`, `zmprov`, `postsuper` |
| Permission errors | `/opt/zimbra/libexec/zmfixperms -e -v` |

---

## 📚 Learn More

- [Zimbra CLI Commands (zmprov)](https://wiki.zimbra.com/wiki/Zmprov)  
- [Stop Admin from Viewing Mail](https://wiki.zimbra.com/wiki/Restrict_Admin_%27View_Mail%27)  
- [Fix Spam Problems](https://sathisharthars.wordpress.com/2013/11/11/managing-spam-mails-in-zimbra-mail-server/)  
- [Get All Mailbox Sizes](https://wiki.zimbra.com/wiki/Get_all_user%27s_mailbox_size_from_CLI)  
- [Zimbra Forums](https://forums.zimbra.org/) — Ask questions!

---

## ✅ Best Practices

| Task | Tip |
|------|-----|
| Change Password | Always notify user |
| Delete Queue | Check ID first → Dry run → Delete |
| New Admins | Use “Restricted Admin Group” — least privilege |
| Spam Attack | Lock → Change Pass → Clear Queue |
| Scripts | Test in staging → Add error checks |
| Logs | Check `/var/log/zimbra.log` daily |

---

## 🎯 Final Tip

> “When Zimbra’s web UI is slow or broken — the command line is your best friend.”


---

> ✍️ **Made by**: Sumon Paul 
> 📅 **Last Updated**: 21-09-2025 
> 🌐 **Works with**: Zimbra 8.8, 9.x, 10.x (Open Source & Network Edition)

---

✅ **You’re now a Zimbra CLI Pro — ready to handle anything from spam to server crashes!**

---

> 🙏 Thank you — your Zimbra server is now safer, faster, and easier to manage.  

---
