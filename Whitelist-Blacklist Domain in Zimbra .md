
# How to whitelist/blacklist domain in zimbra mail server

## yourdomain.com this domain need to add whitelist in zimbra server

> #### Login to your server as root and change to zimbra user

```
su – zimbra
```
### 1 Ways:
#Edit this file /opt/zimbra/conf/salocal.cf.in
```
nano /opt/zimbra/conf/salocal.cf.in
```
```
blacklist_from *fenpropertyservices.co.uk
blacklist_from *marshallsestateagents.co.uk
whitelist_from m.rajesh@yahoo.com
whitelist_from ganesh@gmail.com
```

### 2 Ways: 
#Edit this file /opt/zimbra/conf/amavisd.conf.in
```
nano /opt/zimbra/conf/amavisd.conf.in
```
#Find soft-blacklisting and add the whitelist domain like below

```
# soft-blacklisting (positive score)
'sender@example.net' => 3.0,
'.example.net' => 1.0,
#whitelist domain
'.yourdomain.com' => -10.0,

```
> you need to set the number to -10.0 for whitelist

### Restart Amavis Service
```
zmmtactl restart && zmamavisdctl restart
```
---

## How to Whitelist/Blacklist a domain or email address from Zimbra Webmail

[Reference: how-to-whitelistblacklist-a-domain-or-email-address-from-zimbra-webmail](https://kb.diadem.in/how-to-whitelistblacklist-a-domain-or-email-address-from-zimbra-webmail_1225.html)

---

## Whitelist or Blacklist a domain or email address on Zimbra Amavis

[Ref: zimbra-amavis-spam-filtering]( https://bobcares.com/blog/zimbra-amavis-spam-filtering/)

> 1. First, we create two files that will store the domains and email addresses we wish to whitelist or blacklist.

$ sudo touch /opt/zimbra/conf/{whitelist,blacklist}
All whitelists will be in the file /opt/zimbra/conf/whitelist, and the IPs in the blacklist can be seen in the file /opt/zimbra/conf/blacklist.

Example:

$ cat /opt/zimbra/conf/whitelist
bob@example.com example.org
$ cat /opt/zimbra/conf/blacklist
spammer@example.com
fakedomain.com
After that we modify our /opt/zimbra/conf/amavisd.conf by adding the below lines.

read_hash(%whitelist_sender, '/opt/zimbra/conf/whitelist');
read_hash(%blacklist_sender, '/opt/zimbra/conf/blacklist');
After that, we save the changes and restart the Amavis service.

sudo su - zimbra -c "zmamavisdctl restart"
We can then retry to send emails from a domain/address in the blacklist or the ones in the whitelist.

As a result, we will be able to see that mail delivery is fine now.

---

## How to update a list of trusted MTA networks?
First, we can check the setting for the current list of trustable networks.

$ sudo su - zimbra
$ postconf mynetworks
$ zmprov gs 'zmhostname' zimbraMtaMyNetworks
Next, we can use the following commands to update trustworthy networks in the MTA

$ sudo su - zimbra
$ zmprov ms 'zmhostname' zimbraMtaMyNetworks '127.0.0.0/8 10.0.0.0/8 192.168.3.0/22'

The zmconfigd will automatically restart the MTA processes after this change is made.

---

