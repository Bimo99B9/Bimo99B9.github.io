---
layout: single
title: Sauna - Hack The Box
excerpt: ""
date: 2021-08-07
classes: wide
header:
  teaser: /assets/images/htb-writeup-love/love_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - easy
  - infosec
tags:  
  - Windows
  - Pass-The-Hash
  - wordlists
  - kerbrute
  - impacket
  - smb
---

![](/assets/images/htb-writeup-love/love_logo.png)

Sauna is a very complete Windows machine, in which some of the most common tools are used to gain access and escalate privileges in the system. The enumeration requires making a list of possible usernames using the about page of the website. In the privilege escalation, the Pass-The-Hash technique is used to become administrator, which makes the machine interesting at the same time that it is easy and enjoyable to start with Windows pentesting.

## Portscan

![](/assets/images/htb-writeup-sauna/sauna1.png)

## User enumeration

The Sauna website has an `about.html` page that can be used to make a list of possible usernames, combining names and surnames with some common rules, which can be used to perform user brute-forcing attacks to guess a possible username in the system.

![](/assets/images/htb-writeup-sauna/sauna2.png)

The nmap scan told us the Domain of the Active Directory hosted in the machine. Using the kerbrute tool with the userenum parameter, and providing it with our user list, the IP of the machine and the domain of the machine, the tool will perform a username guessing attack, and report us which usernames are valid usernames, so we can use them later.

In this case, both `sauna`, `administrator` and `fsmith` were valid usernames.

![](/assets/images/htb-writeup-sauna/sauna3.png)


## User shell

Using the GetNPUsers.py tool of impacket, we can get the hash of `fsmith`.

![](/assets/images/htb-writeup-sauna/sauna4.png)

The hash is a kerberos hash, as we can recognize its syntax.

![](/assets/images/htb-writeup-sauna/sauna5.png)

With the help menu of hashcat, we can identify the attack mode necessary to crack the kerberos hash we just obtained. In this case, 18200.

![](/assets/images/htb-writeup-sauna/sauna6.png)

The dictionary used in the attack is rockyou.txt, and in only a second of runtime, we can see that the password is `Thestrokes23`.

![](/assets/images/htb-writeup-sauna/sauna7.png)

Then, we can use the credentials to connect to the machine with evil-winrm.

![](/assets/images/htb-writeup-sauna/sauna11.png)

## User pivoting.

We use winPeas to enumerate the misconfigurations in the machine, and we can find AutoLogon credentials of the user `svc_loanmanager`.

![](/assets/images/htb-writeup-sauna/sauna14.png)

As we did before, it is possible to use the credentials with evil-winrm, so we change to this new user.

![](/assets/images/htb-writeup-sauna/sauna16.png)

## Privilege escalation

We used mimikatz to get the NTLM hash of the administrator user.

![](/assets/images/htb-writeup-sauna/sauna17.png)

This hash can be used to perform a Pass-The-Hash attack, which consists on using an obtained hash to login in the machine, without needing to crack it before. Again, with evil-winrm, it is possible to use the username and the hash to stablish a shell in the machine.

![](/assets/images/htb-writeup-sauna/sauna18.png)