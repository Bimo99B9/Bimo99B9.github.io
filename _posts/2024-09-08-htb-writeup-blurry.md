---
layout: single
title: Blurry - Hack The Box
excerpt: "The blurry machine shows a vulnerability in ClearML, a development suite for ML/DL. It is classified with a medium difficulty."
date: 2024-09-08
classes: wide
header:
  teaser: /assets/images/htb-writeup-blurry/blurry_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - hard
  - infosec
tags:  
  - reverse-engineering
  - ghidra
  - sql-injection

---

![](/assets/images/htb-writeup-blurry/blurry_logo.png)

Blurry is a medium-difficulty Hack The Box machine that highlights a vulnerability in ClearML, a popular ML/DL tool. By exploiting insecure pickle deserialization (CVE-2024-24590) and leveraging misconfigurations, attackers can escalate privileges and gain root access, showcasing real-world risks in machine learning environments.

## System Enumeration

First, an `nmap` is performed to extract the open ports of the system.

![](/assets/images/htb-writeup-blurry/blurry1.png)

Knowing that the open ports were `22` and `80`, this ports are scanned with the `-sCV` flags.

![](/assets/images/htb-writeup-blurry/blurry2.png)

We discover an `nginx 1.18.0`web server running in the port `80`, that redirects to `http://app.blurry.htb/`.

![](/assets/images/htb-writeup-blurry/blurry3.png)

Editing the hosts file, we add the domains of the system to the target IP.

## Web Analysis

The platform has a login to a ClearML portal.

![](/assets/images/htb-writeup-blurry/blurry4.png)

![](/assets/images/htb-writeup-blurry/blurry5.png)

Looking to the Full Name field, we get some usernames, including a "Default User".

![](/assets/images/htb-writeup-blurry/blurry6.png)

When selecting the Default User, it gets automatically logged in. In the workspace, there are some keys.

## Foothold and user access

![](/assets/images/htb-writeup-blurry/blurry7.png)

The `CVE-2024-24590` exploits a pickle file deserialization issue to execute arbitrary code in a ClearML system.

![](/assets/images/htb-writeup-blurry/blurry8.png)

When running the script, a project is requested. Therefore, we can create a project in the ClearML website, or use one of the projects already created.

![](/assets/images/htb-writeup-blurry/blurry9.png)

Despite this, when running the exploit, we get an error because ClearML is not running in our system.

![](/assets/images/htb-writeup-blurry/blurry10.png)

Following the instructions of the web server, we can install ClearML.

![](/assets/images/htb-writeup-blurry/blurry11.png)

The system allows us to create new credentials, which provides several new server subdomains.

![](/assets/images/htb-writeup-blurry/blurry12.png)

Again, we add these domains to the `etc/hosts/` file.

![](/assets/images/htb-writeup-blurry/blurry13.png)

With these changes, it is possible to init ClearML in our system with `clearml-init`.

![](/assets/images/htb-writeup-blurry/blurry14.png)

Running the script while listening on a port for incomming connections, allows us to get a reverse shell with user access, under the `jippity` user.

## Privilege escalation

This user can run the command `/usr/bin/evaluate_model /models/*.pth` as root.

![](/assets/images/htb-writeup-blurry/blurry15.png)

First, I copied the `id_rsa` to have SSH connection to the system without the reverse shell.

![](/assets/images/htb-writeup-blurry/blurry16.png)

Assigning the correct permissions and connecting to the machine with SSH makes the shell more stable for further exploitation.

![](/assets/images/htb-writeup-blurry/blurry17.png)

The privilege escalation is done using a malicious pytorch model that, when run, executes a connection with `nc` to our host. As this is run as `root`, it allows us to get a reverse shell with root access to the system. 

![](/assets/images/htb-writeup-blurry/blurry18.png)

The next step is to run the `evaluate_model` binary, which will "evaluate", and therefore run, our malicious model, while we listen in the specified port for the incoming connection.

![](/assets/images/htb-writeup-blurry/blurry19.png)

This gets us the root shell, and the root flag!

![](/assets/images/htb-writeup-blurry/blurry20.png)

## Summary

This has been an easy-medium challenge, that showcases how a very common ML/DL tool can be exploited, from a simple deserialization issue, to obtain root access to the server.

![](/assets/images/htb-writeup-blurry/blurry21.png)