---
layout: post
title: Exfiltrating Data via Wget though a Blind Command injection
date: 05-04-2025
categories: [Techniques]
tags: [exfiltration, blind, commandinjection, wget]
image: "https://cdn.prod.website-files.com/642adcaf364024654c71df23/6769c0820cf95b3bdf630c2c_blog-visuals-state-of-command-injection-thumbnail-1_72f06fa981fd18b69cb65a075bae0a83.png"
---
## Introduction
Hello, I’m cyb4x, a Red Team enthusiast with a focus on offensive security tactics. In this article, we’ll explore one such method: exfiltrating data using wget through blind command injection vulnerabilities. When a target system reveals a command injection flaw, it opens the door to remote code execution (RCE) without the need for complex exploits. I’ll walk you through the process of using wget to locate and extract valuable files, navigate the filesystem, and even access database contents, all while flying under the radar.

## Blind Command Injection
### What is Blind Command Injection?
Blind command injection is a security vulnerability that allows an attacker to execute arbitrary commands on a server, despite not receiving direct feedback or output from those commands. Unlike standard command injection, where the attacker sees the results of each command they execute, blind command injection requires creativity and patience, as there’s no immediate confirmation that a command succeeded or failed. Attackers often rely on side effects, such as delays or HTTP responses, to infer the success of their actions, making it challenging but achievable to navigate and exploit.

### Identifying Command Injection (Blind) Vulnerabilities
Command injection vulnerabilities typically arise when an application improperly handles user input, passing it directly into a command executed on the operating system without proper sanitization. An attacker might identify this type of vulnerability by testing inputs with symbols like ;, &&, or |, which can chain or initiate additional commands. If the application is vulnerable to blind injection, it may not display the output, but other observable indicators like server delays or altered responses may hint that commands are being processed in the background. Careful testing, including trial commands that generate a detectable result (e.g., adding a delay with sleep), can help to confirm the existence of blind injection.

> Coming soon!
{: .prompt-tip }
