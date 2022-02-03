---
layout: single
title: Monitors - Hack The Box
excerpt: "Monitors is a hard machine that goes from apache, cacti, to wordpress servers. Privilege escalation involves lateral movement between users and escaping from a docker image, which makes this machine a long and difficult challenge, but very entertaining for somewhat experienced attackers."
date: 2021-10-10
classes: wide
header:
  teaser: /assets/images/htb-writeup-monitors/monitors_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - hard
  - infosec
tags:  
  - Wordpress
  - Cacti
  - SQL injection
  - LFI
  - Docker
  - Tomcat
---

![](/assets/images/htb-writeup-monitors/monitors_logo.jpg)

Monitors is a hard machine that goes from apache, cacti, to wordpress servers. Privilege escalation involves lateral movement between users and escaping from a docker image, which makes this machine a long and difficult challenge, but very entertaining for somewhat experienced attackers.

## Nmap

The port scan only reveals SSH and an Apache web server.

![](/assets/images/htb-writeup-monitors/monitors1.png)

## Wordpress

This web page in the port 80, is powered by Wordpress, as we can see on the bottom right.

![](/assets/images/htb-writeup-monitors/monitors2.png)

The version can be retrieved searching `generator` in the source code. This wordpress is running the `5.5.1` version.

![](/assets/images/htb-writeup-monitors/monitors3.png)

If we enumerate directories, we'll find the common ones on a wordpress site.

![](/assets/images/htb-writeup-monitors/monitors4.png)

In the plugins folder there is only one plugin, `wp-with-spritz`.

![](/assets/images/htb-writeup-monitors/monitors5.png)

The wpscan tool reveals that this plugin is vulnerable to Unauthenticated File Inclusion.

![](/assets/images/htb-writeup-monitors/monitors6.png)

This exploit has a proof of concept, and we can see that it is really easy to exploit it.

![](/assets/images/htb-writeup-monitors/monitors8.png)

## Enumeration

Now, we can read files in the system, as the `/etc/passwd`. Here we discover the `marcus` user.

![](/assets/images/htb-writeup-monitors/monitors9.png)

With burpsuite we can use this exploit way easier. The hostname of the machine is `monitors`.

![](/assets/images/htb-writeup-monitors/monitors10.png)

In the `wp-config.php` there are some credentials. `wpadmin:BestAdministrator@2020!`.

![](/assets/images/htb-writeup-monitors/monitors12.png)

We have found many things, but there is nothing really useful that we can do with this. Therefore, we will be using a local file inclusion linux wordlist.

![](/assets/images/htb-writeup-monitors/monitors13.png)

With burpsuite, we can set the payload and run this wordlists on our target.

![](/assets/images/htb-writeup-monitors/monitors14.png)

This wordlists was useful, as we found content in the `/proc/self/fd/10` path.

![](/assets/images/htb-writeup-monitors/monitors15.png)

There we have information about `cacti`. Another thing to take into account. Let's continue.

![](/assets/images/htb-writeup-monitors/monitors16.png)

Searching about useful files, I found a config file I didn't have checked, `000-default.conf`.

![](/assets/images/htb-writeup-monitors/monitors17.png)

In this file, we finally find a host, so there is virtual hosting in this machine. `cacti-admin.monitors.htb` was the host of the cacti web server we found before.

![](/assets/images/htb-writeup-monitors/monitors18.png)

## Cacti

The main page asks us for a login input.

![](/assets/images/htb-writeup-monitors/monitors19.png)

Testing with the credentials I had, I found that the `admin:BestAdministrator@2020!` was working.

![](/assets/images/htb-writeup-monitors/monitors20.png)

The dashboard is the normal Cacti page. We can see the version in the top right.

![](/assets/images/htb-writeup-monitors/monitors21.png)

In this version there is a SQL injection vulnerability in the `export` button when editing colors.

![](/assets/images/htb-writeup-monitors/monitors23.png)

If we edit the colors in the `color.php` page, we can see the `export` button.

![](/assets/images/htb-writeup-monitors/monitors24.png)

Let's intercept that request with burpsuite, and modify it to exploit the SQL injection.

![](/assets/images/htb-writeup-monitors/monitors25.png)

We've found a hash. With hashcat, its mode is `3200`. We can try to crack it.

![](/assets/images/htb-writeup-monitors/monitors26.png)

The password was easily cracked, but it was the one we already had.

![](/assets/images/htb-writeup-monitors/monitors27.png)

It doesn't matter as we can get a shell with the SQL injection with this available exploit for `Cacti 1.2.12`.

![](/assets/images/htb-writeup-monitors/monitors28.png)

With the exploit and the credentials, and a nc listener in another window, we get a shell.

![](/assets/images/htb-writeup-monitors/monitors29.png)

The shell belongs to the `www-data` user.

![](/assets/images/htb-writeup-monitors/monitors30.png)

We can upgrade our shell with `python3 -c "import pty;pty.spawn('/bin/bash')";`

![](/assets/images/htb-writeup-monitors/monitors31.png)

## Lateral movement

As we saw before, there is a user in the machine called `marcus`. We'd like to move into this user. With `grep 'marcus' /etc -R 2>/dev/null` we can search for marcus in the system.

![](/assets/images/htb-writeup-monitors/monitors32.png)

In the `/home/marcus/.backup/backup.sh` there are some credentials.

![](/assets/images/htb-writeup-monitors/monitors33.png)

We can use these credentials to connect with ssh as the marcus user.

![](/assets/images/htb-writeup-monitors/monitors34.png)

## Privilege escalation

Now we have the user part of the machine, but here the machine becomes hard. Until now, it was not really difficult to find hints and exploit the vulnerabilities. The first thing we find as marcus is a To-do note in the home directory, refering to a docker image pending to update.

![](/assets/images/htb-writeup-monitors/monitors36.png)

With `netstat -tunlp` we find an interesting port.

![](/assets/images/htb-writeup-monitors/monitors37.png)

This port was commonly used by HTTPS applications.

![](/assets/images/htb-writeup-monitors/monitors38.png)

Then, we reconnect as marcus, port forwarding the port with SSH. ``ssh -L 8443:127.0.0.1:8443 -R 4444:127.0.0.1:4444 -R 8080:127.0.0.1:8080 marcus @monitors.htb`

## Apache

![](/assets/images/htb-writeup-monitors/monitors39.png)

In that port there is an Apache Tomcat 9.0.31 running.

![](/assets/images/htb-writeup-monitors/monitors40.png)

We can read about the vulnerabilities fixed in the newer versions to discover some important things.

![](/assets/images/htb-writeup-monitors/monitors42.png)

This version is affected by the `CVE-2020-9484`.

![](/assets/images/htb-writeup-monitors/monitors43.png)

I don't like metasploit, but this is the best way to exploit this particular CVE, as we have some modules that match very well what we want to do.

![](/assets/images/htb-writeup-monitors/monitors44.png)

If we set the options and the payload to get a reverse shell and run the exploit, we'll end up in a docker image.

![](/assets/images/htb-writeup-monitors/monitors45.png)

## Docker

The first step is to upgrade the shell.

![](/assets/images/htb-writeup-monitors/monitors46.png)

Now, we can start enumerating. We find that a Linux capability `SYS_MODULE` is enabled with the `capsh --print` command.

![](/assets/images/htb-writeup-monitors/monitors47.png)

I found an article that explained really well how to exploit this capability to get out of a docker image.

![](/assets/images/htb-writeup-monitors/monitors48.png)

With this code, we can send a reverse shell to our attacking machine. The point is that this shell is root in the main system, not in the docker image.

![](/assets/images/htb-writeup-monitors/monitors50.png)

First, we write the code in our own machine, and adapt it to get the shell in our listening nc instance.

![](/assets/images/htb-writeup-monitors/monitors53.png)

In the docker, we get the code with an http server in our system. If we follow the steps of the article, we have to compile the file, and run `insmod reverse-shell.ko` to trigger the reverse shell.

![](/assets/images/htb-writeup-monitors/monitors54.png)

In our machine we get the shell as some many attempts, and see that we are root in the monitors machine. We have rooted the box.

![](/assets/images/htb-writeup-monitors/monitors55.png)

Now, we can retrieve the flag and submit it in HackTheBox.

![](/assets/images/htb-writeup-monitors/monitors56.png)