---
layout: single
title: Remote - Hack The Box
excerpt: "Remote is a Windows machine with the Umbraco web content manager, which is exploited through a mountable partition and cached credentials whose greatest vulnerability is an outdated version of Umbraco, what makes possible to exploit the machine."
date: 2020-10-13
classes: wide
header:
  teaser: /assets/images/htb-writeup-remote/remote_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - easy
  - infosec
tags:  
  - Windows
  - umbraco
  - msfvenom
  - teamviewer
  - evil-winrm
---

![](/assets/images/htb-writeup-remote/remote_logo.jpg)

Remote is a Windows machine with the Umbraco web content manager, which is exploited through a mountable partition and cached credentials whose greatest vulnerability is an outdated version of Umbraco, what makes possible to exploit the machine.

## Portscan

![](/assets/images/htb-writeup-remote/remote1.png)

## Foothold

Using the script `nfs-showmount` of nmap, we discover a mountable folder `/site_backups`

![](/assets/images/htb-writeup-remote/remote2.png)

When we mount the folder, we have the files of a backup of what it seems to be an Umbraco site.

![](/assets/images/htb-writeup-remote/remote3.png)

In the `/App_Data/` folder, there is one of the key files in an Umbraco system: `Umbraco.sdf`.

![](/assets/images/htb-writeup-remote/remote6.png)

If we filter the word `admin` in that file, we find a SHA-1 hash.

![](/assets/images/htb-writeup-remote/remote7.png)

Decrypted, this hash was a password of the admin user, which we will use later.

![](/assets/images/htb-writeup-remote/remote9.png)

## Website

The website is an umbraco login page, without register option, so some credentials are needed to log in the site. (As SQL injections are not working in any of the parameters.)

![](/assets/images/htb-writeup-remote/remote5.png)

In the `Web.config` file of the mounted folder, we can see that the used version is `7.12.4`.

![](/assets/images/htb-writeup-remote/remote4.png)

That version is vulnerable to RCE (Authenticated), exploit that we can use as we now have valid credentials.

![](/assets/images/htb-writeup-remote/remote10.png)

Then, we clone the repository and install the Python requirements.

![](/assets/images/htb-writeup-remote/remote11.png)

If we login to the website with the credentials that we have, we will see an option to upload files to the web. With a bit of testing, we can see that the server doesn't check the file that you upload.

![](/assets/images/htb-writeup-remote/remote13.png)

## Reverse shell

With that known, we can create a .exe payload using msfvenom, which we will upload and then use our RCE exploit to execute it and gain access to the machine.

![](/assets/images/htb-writeup-remote/remote12.png)

Now, we upload the file to the web.

![](/assets/images/htb-writeup-remote/remote14.png)

Then, we test that we have RCE in the machine with our exploit, and use it to find in the directories the path of our uploaded payload.

![](/assets/images/htb-writeup-remote/remote15.png)

The path of the file is `C:/inetpub/wwwroot/Media/1034/hello.exe`. We can execute it with the powershell.

![](/assets/images/htb-writeup-remote/remote19.png)

This time we will use metasploit to stablish the reverse shell, as it could be useful later to have our shell in this environment.

![](/assets/images/htb-writeup-remote/remote20.png)

With the command `shell`, the reverse shell is launched and we now have access to the system.

![](/assets/images/htb-writeup-remote/remote21.png)

At this point the user.txt flag can be taken.

![](/assets/images/htb-writeup-remote/remote22.png)

## System enumeration

We will be enumerating the system via Winpeas. To run it we will upload it as we did with the payload before.

![](/assets/images/htb-writeup-remote/remote26.png)

In the enumeration we find that TeamViewer Version 7 is installed in the machine.

![](/assets/images/htb-writeup-remote/remote27.png)

TeamViewer is vulnerable as we can get the cached credentials with metasploit, and the passwords may be the same that the privileged user of the system.

![](/assets/images/htb-writeup-remote/remote28.png)

As we now have a user `Administrator` and a password `!R3m0te!`, we can log in to the machine using evil-winrm and take the root.txt flag.

![](/assets/images/htb-writeup-remote/remote29.png)

