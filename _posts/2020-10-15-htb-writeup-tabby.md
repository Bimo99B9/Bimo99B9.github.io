---
layout: single
title: Tabby - Hack The Box
excerpt: "Tabby is an easy and enjoyable Linux machine with an LFI to get the foothold, lateral movement from tomcat and an abuse of the lxd group privilege."
date: 2020-10-15
classes: wide
header:
  teaser: /assets/images/htb-writeup-tabby/tabby_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:  
  - Linux
  - tomcat
  - lfi
  - lxd
  - fcrackzip
---

![](/assets/images/htb-writeup-tabby/tabby_logo.jpg)

Tabby is an easy and enjoyable Linux machine with an LFI to get the foothold, lateral movement from tomcat and an abuse of the lxd group privilege.

## Portscan

![](/assets/images/htb-writeup-tabby/tabby1.png)

## Foothold

We have only 3 open ports. SSH, HTTP and HTTPS. The HTTPS one, 8080, is a Tomcat website that asks for credentials.

![](/assets/images/htb-writeup-tabby/tabby2.png)

In the HTTP website, there is a page vulnerable to LFI, as it receives a parameter in the URL indicating the file to read, and does not sanitize the input, so we can read the /etc/passwd file and find **ash** and **tomcat** users.

![](/assets/images/htb-writeup-tabby/tabby3.png)

We know that tomcat is running in another port, so it is possible to read the configuration file with the Local File Inclusion, and get the credentials.

![](/assets/images/htb-writeup-tabby/tabby4.png)

## Tomcat War Deployer

With valid tomcat credentials, we can upload a payload and execute it to get a reverse shell. In the case of tomcat, this can be easily done with the [Tomcat War Deployer Script](https://github.com/mgeeky/tomcatWarDeployer).

![](/assets/images/htb-writeup-tabby/tabby5.png)

## Lateral movement

Enumerating the files in the system we find a zipped backup.

![](/assets/images/htb-writeup-tabby/tabby9.png)

We can copy this zip to our local machine to unzip it there.

![](/assets/images/htb-writeup-tabby/tabby10.png)

With fcrackzip we easily find that the password is `admin@it`, so we don't need to extract the hash and use john or hashcat.

![](/assets/images/htb-writeup-tabby/tabby12.png)

## Privilege escalation

As the found credential is the one that the user `ash` uses, we can log in to the machine as `ash` and get the user flag.

If we execute the command `id` we will see that we are member of the group lxd, which is a vulnerability that we can use to escalate privileges. (See `searchsploit lxd`).

Then, the steps are:

* Clone the alpine container image from the repository

![](/assets/images/htb-writeup-tabby/tabby13.png)

* Move it to the victim machine, starting a python server in your machine `python3 -m http.server 8000`.

![](/assets/images/htb-writeup-tabby/tabby14.png)

* Execute the payload, and get the root flag.

![](/assets/images/htb-writeup-tabby/tabby15.png)