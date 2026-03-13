# 🚩 [smol] — TryHackMe Write-Up

> **Platform:** TryHackMe  
> **Room:** [Room Link](https://tryhackme.com/room/smol)  

---

## 📋 Table of Contents

1. [Overview](#-overview)
2. [Reconnaissance](#-reconnaissance)
3. [Enumeration](#-enumeration)
4. [Exploitation](#-exploitation)
5. [Privilege Escalation](#-privilege-escalation)

---

## 🔭 Overview

>At the heart of **Smol** is a WordPress website, a common target due to its extensive plugin ecosystem. The machine showcases a publicly known vulnerable plugin, highlighting the risks of neglecting software updates and security patches. Enhancing the learning experience, Smol introduces a backdoored plugin, emphasizing the significance of meticulous code inspection before integrating third-party components.

**Objective:** [e.g., Compromise a vulnerable WordPress server and escalate to root]

**Skills Used:**

- [e.g., Port Scanning with Nmap]
- [e.g., Web Application Exploitation]
- [e.g., Linux Privilege Escalation]
- [e.g., Password Cracking]

---

## 🔍 Reconnaissance

### Nmap Scan
We found two open ports and we will check the web application.

![[smol room/images/1nmap.png]]

---

## 🔎 Enumeration

### Web Enumeration
Using *gobuster*, we found some directories that redirect to *www.smol.thm*, so we added this URL to */etc/hosts*
![[2gobuster.png]]

From these directories we discovered a *WordPress* server, so we used *wpscan* to enumerate plugins, users, and themes for any credentials or vulnerable plugins.

![[3wpscan.png]]

We found a vulnerable plugin called *JSmol* that has two CVEs (XSS, SSRF).

---

## 💥 Exploitation

After our research, we discovered that we can exploit this SSRF vulnerability to get an interesting file called *wp-config* containing some information and credentials via *LFI*.
```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```

![[5wpconfig.png]]

 We found credentials that we can use in the *wp-login.php* page.
 ![[6login.png]]

After some crawling on this site, we found an interesting page.


![[7page.png]]


![[7page.png]]
 This page contains information about a common plugin called *Hello Dolly*.
 ![[8hellodollyplugin.png]]
 Now, we wanted to know the PHP file of this plugin, so we did a quick search and discovered its name and path.
 ![[9hellophplocation.png]]

 We then used the previous vulnerability *LFI* to get the *hello.php* file.
 ```
 http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../hello.php

 ```
 
![[10base64.png]]
We discovered a *hello_dolly* function that has encoded content, so we decoded it using *CyberChef*.
![[11decode.png]]

It decoded to *escapestring*, so we decoded it further to plain text.
![[12escapestring.png]]

We found a *cmd* parameter, and after some research we learned that to use this function we need to be on the wp-admin page, so we navigated to this link and tried to execute a command.
```
http://www.smol.thm/wp-admin/index.php?cmd=whoami
```

![[13cmdwhoami.png]]

Here we can execute shell command and after trying many ways using https://www.revshells.com/ , we got reverse shell with *netcat* with this payload:
![[14revshell.png]]
We got a reverse shell, but we didn't have permission to access any user's files. However,
using *wappalyzer* extension we found MySQL database service, so we can access the databases 
![[15sql1.png]]
we found the *wp_users* table and I guessed that it contains user credentials,
and that was right.
 ![[16sql2.png]]
 we collected the users with their hashes to crack them with *john and ripper*
 
 ![[17hashes.png]]
 It only cracked one hash for user named *diego*
 ![[18diego.png]]
 Now we have this user and his password, so we can switch to him with *su diego*, and when we listed his files we got the *user.txt* flag.
 ![[19.1 su diego.png]]
Now we want to escalate our privilege to be root 
 

---
## ⬆️ Privilege Escalation
The current user is *diego*, but this user doesn't have permission to run sudo. We ran *linpeas.sh* but didn't find anything interesting, so we tried to access other users' directories. We managed to access the directory of a user named *think* and found his SSH private key, which we can use to log in as that user.
![[19.2 diego access on think dir.png]]

![[20ssh think and zip file.png]]

After logging in as this user, we found a zip file but couldn't unzip it because it belongs to a user named *gege*.
 Now we want to access as *gege* we tried *su gege* and it worked , we can switch to *gege* from *think* and now we will unzip this file but it has a password that we don't have
![[22fail unzip.png]]

Now we only have one way to get this file on our machine by upload this file to python server and download it on our machine by *wget* 
![[23 python server.png]]

![[24wget.png]]
Then we have to convert this zip file to hash password because it's a formula can *john and ripper* tool can understand by *zip2john* tool
![[25zip2john.png]]

Now we can crack this hash with *john and ripper*
![[26zip pass.png]]
After access this directory we got *wp-config* file which has interesting credentials
![[27 wp-config xavi.png]]
We got user named *xavi* and his password which can access with these credentials then we tried to show his sudo permissions with *sudo -l* as we tried with all previous users 
![[28 xavi sudo.png]]
This time we discovered that xavi can execute all sudo commands as a root so, we can now access root pass and get *root.txt* flag
![[29 root flag.png]]

---



## Write-up by *Ahmed Essam*