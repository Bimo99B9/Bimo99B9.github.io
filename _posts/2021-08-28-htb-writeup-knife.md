---
layout: single
title: Knife - Hack The Box
excerpt: "The Knife machine of HackTheBox is an easy Linux machine very useful to understand basic concepts about enumeration, and how to stablish a simple reverse shell. It is also helpful to understand the escalation of privileges using Gtfobins."
date: 2021-08-28
classes: wide
header:
  teaser: /assets/images/htb-writeup-knife/knife_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - easy
  - infosec
tags:  
  - Linux
  - PHP
  - GTFObins
  - knife
---

![](/assets/images/htb-writeup-knife/knife_logo.png)

The Knife machine of HackTheBox is an easy Linux machine very useful to understand basic concepts about enumeration, and how to stablish a simple reverse shell. It is also helpful to understand the escalation of privileges using Gtfobins.

## Nmap

![](/assets/images/htb-writeup-knife/knife1.png)

Only the ports 22 (SSH) and 80 (HTTP) are open.

## Website

In the website we will see a simple page with nothing interesting at a glance.

![](/assets/images/htb-writeup-knife/knife2.png)

Using Wappalyzer, a web enumeration tool, we find that the web is running `PHP 8.1.8`, which is a deprecated version that may have some vulnerabilities available.

## PHP vulnerability

![](/assets/images/htb-writeup-knife/knife3.png)

With the command `searchsploit PHP 8.1.8` we can find the exploits that affect this PHP version.

![](/assets/images/htb-writeup-knife/knife4.png)

The most interesting one is the one about a Remote Code Injection backdoor, so we download it.

![](/assets/images/htb-writeup-knife/knife6.png)

In its github we can see its syntax, which is `python3 backdoor.py -u (IP) -c (command)`
Then, we try to list the files and directories in the machine with this exploit, and we check that it is working propperly.

![](/assets/images/htb-writeup-knife/knife9.png)

## Reverse shell

As we have RCE, we will execute a Reverse Shell bash command available at [GTFObins](https://gtfobins.github.io/) to get a shell.

![](/assets/images/htb-writeup-knife/knife11.png)

Then, if we listen in the selected port with netcat using `nc -nvlp 4444`, we get a shell as the user `james`.

![](/assets/images/htb-writeup-knife/knife12.png)

## Knife command exploit

If we run `sudo -l` to list the commands that we can execute as a privileged user, we see that `knife` can be sudo executed.

![](/assets/images/htb-writeup-knife/knife13.png)

I didn't know what this command does, so I checked the help page, and realized that there was an `exec` parameter to execute scripts.

![](/assets/images/htb-writeup-knife/knife14.png)

![](/assets/images/htb-writeup-knife/knife15.png)

Then, as the knife command is executed as sudo, we can execute the bash itself to obtain access to a root shell directly and easy.

![](/assets/images/htb-writeup-knife/knife16.png)

We can now get the `root.txt` flag.

![](/assets/images/htb-writeup-knife/knife17.png)