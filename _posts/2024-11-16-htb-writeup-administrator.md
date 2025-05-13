---
layout: single
title: Administrator - Hack The Box
excerpt: "Exploring Active Directory vulnerabilities to achieve domain administrator access."
date: 2024-11-16
classes: wide
header:
  teaser: /assets/images/htb-writeup-administrator/admin_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
tags:  
  - Active Directory
  - Windows Server
  - Privilege Escalation
  - Penetration Testing

---

![](/assets/images/htb-writeup-administrator/admin_logo.png)

# Overview

This writeup showcases the exploitation of the **Administrator** machine on Hack The Box. The target system is a Windows Server 2022 Domain Controller, highlighting vulnerabilities in Active Directory configurations and mismanaged credentials.

# Nmap Enumeration

An initial Nmap scan was performed to identify open ports and services:

```bash
nmap -p- -T5 --open -vv -n -Pn 10.10.11.42 -oG ports
```

Open ports included:

- FTP: 21/tcp
- Domain Services: 53/tcp
- Kerberos: 88/tcp
- SMB: 445/tcp
- LDAP: 389/tcp
- WinRM: 5985/tcp

![](/assets/images/htb-writeup-administrator/admin1.png)

A targeted scan was then executed for detailed service and version information:

```bash
nmap -sCV -p21,53,88,135,139,389,445,593,636,3268,3269,5985,9389,47001 10.10.11.42 -oN targeted
```

Findings confirmed the server as a Windows Domain Controller with services like Kerberos, SMB, and LDAP configured:

![](/assets/images/htb-writeup-administrator/admin4.png)

# Initial Access

Using `crackmapexec` with discovered credentials `Olivia:ichliebedich`, it was possible to verify domain access:

![](/assets/images/htb-writeup-administrator/admin5.png)

With these credentials, `evil-winrm` provided initial shell access. Tools like Mimikatz, BloodHound, and SharpHound were found in Olivia's documents folder:

![](/assets/images/htb-writeup-administrator/admin8.png)

# Privilege Enumeration

Running `whoami /priv` revealed Olivia's privileges, including `SeMachineAccountPrivilege`, allowing the addition of workstations to the domain. Account details were checked with:

```bash
net user Olivia
```

![](/assets/images/htb-writeup-administrator/admin9.png)

# Active Directory Enumeration

## SharpHound Collection
Using `SharpHound.exe -c all`, data about domain mappings and group memberships was collected for further analysis:

![](/assets/images/htb-writeup-administrator/admin10.png)

## BloodHound Analysis
Data was analyzed with BloodHound to identify privilege escalation paths. Using `netexec`, additional enumeration was performed, and findings were exported:

![](/assets/images/htb-writeup-administrator/admin14.png)

## RPC Enumeration
Using `rpcclient`, domain users and groups were enumerated. Key users like `Administrator` and `michael` were identified:

![](/assets/images/htb-writeup-administrator/admin15.png)

# Kerberoasting Michael

Time was synchronized with the domain controller using `rdate` to ensure Kerberos attacks work properly:

![](/assets/images/htb-writeup-administrator/admin19.png)

Using `GetUserSPNs.py -request`, a Kerberos hash for the user `michael` was extracted and cracked with `hashcat`:

```bash
hashcat -m 13100 michael.hash rockyou.txt
```

Password retrieved: `password123`.

![](/assets/images/htb-writeup-administrator/admin21.png)

# Accessing Michael's Account

With the cracked password, logged in as Michael using `evil-winrm` and verified privileges:

![](/assets/images/htb-writeup-administrator/admin22.png)

Analyzing Active Directory permissions revealed significant control over key objects:

![](/assets/images/htb-writeup-administrator/admin23.png)

# Exploiting Benjamin's Account

Michael's privileges were used to reset Benjamin's password via `net rpc`. The new password allowed login via WinRM:

![](/assets/images/htb-writeup-administrator/admin25.png)

# FTP Discovery

Logged into the FTP service using Benjamin's credentials and retrieved a file `Backup.psafe3`. The file was cracked using `john` with `rockyou.txt`:

Password retrieved: `tequieromucho`.

![](/assets/images/htb-writeup-administrator/admin28.png)

# Using Password Safe Credentials

Using `pwsafe`, additional credentials for users `alexander`, `emily`, and `emma` were extracted. Emily's credentials enabled login via WinRM:

![](/assets/images/htb-writeup-administrator/admin31.png)

# Privilege Escalation to Ethan

Emily's permissions allowed modification of Ethan's account. Using `pywhisker.py`, a certificate was created and exploited to retrieve Ethan's credentials:

![](/assets/images/htb-writeup-administrator/admin33.png)

Ethan's Kerberos hash was cracked to retrieve the password: `Limpbizkit`.

![](/assets/images/htb-writeup-administrator/admin35.png)

# Domain Admin Privileges

Ethan's credentials were used with `secretsdump.py` to extract NTDS.dit secrets, including the Administrator's NTLM hash:

![](/assets/images/htb-writeup-administrator/admin37.png)

Using the Administrator's hash with `evil-winrm`, full control of the domain was achieved:

![](/assets/images/htb-writeup-administrator/admin38.png)

# Conclusion

Successfully exploited the **Administrator** machine, gaining domain administrator privileges. A rewarding challenge in Active Directory exploitation. 🎉

![](/assets/images/htb-writeup-administrator/admin39.png)
