---
layout: single
title: Love - Hack The Box
excerpt: "Love is a very easy Windows machine that can be easily solved if some basic concepts are clear. It contains a SSRF attack, which is not very common and this machine is a very good example of how it works. Enumeration is also very important here, for both foothold and privilege escalation, this last one taking advantage of the AlwaysInstallElevated feature being turned on."
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
  - 
---

![](/assets/images/htb-writeup-love/love_logo.png)

Love is a very easy Windows machine that can be easily solved if some basic concepts are clear. It contains a SSRF attack, which is not very common and this machine is a very good example of how it works. Enumeration is also very important here, for both foothold and privilege escalation, this last one taking advantage of the AlwaysInstallElevated feature being turned on.

## Nmap