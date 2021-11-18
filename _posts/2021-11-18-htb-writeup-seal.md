---
layout: single
title: Seal - Hack The Box
excerpt: ""
date: 2021-11-18
classes: wide
header:
  teaser: /assets/images/htb-writeup-seal/seal_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - medium
  - infosec
tags:  
  - fuzzing
  - Tomcat
  - ansible

---

![](/assets/images/htb-writeup-seal/seal_logo.jpg)

Seal

## Nmap

The nmap scan reveals different http ports.

![](/assets/images/htb-writeup-seal/seal1.png)

One of them with interesting information.

![](/assets/images/htb-writeup-seal/seal2.png)

We can add the host to out `/etc/hosts` file.

![](/assets/images/htb-writeup-seal/seal4.png)

## Gitbucket

Let's use the Set-Cookie with the 8080 port.

![](/assets/images/htb-writeup-seal/seal5.png)

There is a login page.

![](/assets/images/htb-writeup-seal/seal6.png)

As there is a register page, I decided to register.

![](/assets/images/htb-writeup-seal/seal7.png)

We can see some commits.

![](/assets/images/htb-writeup-seal/seal8.png)

In one of them there are tomcat credentials.

![](/assets/images/htb-writeup-seal/seal9.png)

If we keep searching, there is something about mutual authentication enabled.

![](/assets/images/htb-writeup-seal/seal10.png)

The credentials with the `Alex` username don't work.

![](/assets/images/htb-writeup-seal/seal11.png)

There is a "Core Dev" named Luis.

![](/assets/images/htb-writeup-seal/seal12.png)

If we fuzz the host with gobuster we can find the `/manager` path.

![](/assets/images/htb-writeup-seal/seal13.png)

In the `/manager` path there is also a `/status` directory.

![](/assets/images/htb-writeup-seal/seal14.png)

If we go there, we'll find a login form.

![](/assets/images/htb-writeup-seal/seal15.png)

The credentials previously found are valid.

![](/assets/images/htb-writeup-seal/seal20.png)

## Tomcat

It is a tomcat manager page.

![](/assets/images/htb-writeup-seal/seal21.png)

We can use path traversal via reverse proxy mapping. Information in the image below.

![](/assets/images/htb-writeup-seal/seal22.png)

And now we are in the html directory

![](/assets/images/htb-writeup-seal/seal23.png)

We can exploit it as every tomcat manager, with a WAR exploit.

![](/assets/images/htb-writeup-seal/seal24.png)

The payload can be created with msfvenom.

![](/assets/images/htb-writeup-seal/seal25.png)

The upload is forbidden.

![](/assets/images/htb-writeup-seal/seal26.png)

Let's intercept the request with burpsuite.

![](/assets/images/htb-writeup-seal/seal27.png)

We have to use the path traversal.

![](/assets/images/htb-writeup-seal/seal28.png)

Now we have the shell in the manager.

![](/assets/images/htb-writeup-seal/seal29.png)

If we listen in the port with netcat and trigger the shell from the tomcat manager, we get a shell.

![](/assets/images/htb-writeup-seal/seal30.png)

Let's upgrade it with Python

![](/assets/images/htb-writeup-seal/seal31.png)

Now we have a propper shell as the tomcat user.

![](/assets/images/htb-writeup-seal/seal32.png)

## Enumeration

As `tomcat` we can't take the flag, we need the `luis` user.

![](/assets/images/htb-writeup-seal/seal33.png)

We have read permissions, and in its home directory there is a hidden `.ansible` folder.

![](/assets/images/htb-writeup-seal/seal34.png)

This is what I found about Ansible.

![](/assets/images/htb-writeup-seal/seal35.png)

It has ansible playbook installed.

![](/assets/images/htb-writeup-seal/seal36.png)

The run.yml file has `copy_links` set to yes, which can be easily exploited.

![](/assets/images/htb-writeup-seal/seal37.png)

Creating some symbolic links with `ln` we can get the rsa keys.

![](/assets/images/htb-writeup-seal/seal39.png)

If we unzip it we get the keys.

![](/assets/images/htb-writeup-seal/seal40.png)

Here it is the private key.

![](/assets/images/htb-writeup-seal/seal42.png)

With the propper rights we can use it to stablish a SSH connection as `luis`.

![](/assets/images/htb-writeup-seal/seal43.png)

## Privilege escalation

Now, with `sudo -l` we can see that as `luis` we can run `ansible-playbook` as root.

![](/assets/images/htb-writeup-seal/seal45.png)

If we search the binary in GTFObins, we will find a really easy way of getting a root shell exploiting the privileged binary.

![](/assets/images/htb-writeup-seal/seal46.png)

Following the steps, we are finally root.

![](/assets/images/htb-writeup-seal/seal47.png)