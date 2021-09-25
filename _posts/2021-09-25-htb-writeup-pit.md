---
layout: single
title: Pit - Hack The Box
excerpt: "Pit is a medium HackTheBox machine that targets SNMP exploitation and enumeration. It is enumerated with the public community, and an attack to SeedDMS gives us RCE to gain access to a CentOS control pannel. Some misconfigurations in a bash script which works with SNMP are used to escalate privileges and root this quite complex system."
date: 2021-09-25
classes: wide
header:
  teaser: /assets/images/htb-writeup-pit/logo_pit.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - medium
  - infosec
tags:  
  - SNMP
  - UDP
  - CentOS
  - Nginx
  - SELinux
---

![](/assets/images/htb-writeup-pit/logo_pit.png)

Pit is a medium HackTheBox machine that targets SNMP exploitation and enumeration. It is enumerated with the public community, and an attack to SeedDMS gives us RCE to gain access to a CentOS control pannel. Some misconfigurations in a bash script which works with SNMP are used to escalate privileges and root this quite complex system.

## Nmap - TCP

The first nmap scan reveals an OpenSSH service in the default port 22, and two web services. The first one, in the port 80, and the second one, with a commonName `dms-pit.htb`.

![](/assets/imageshtb-writeup-pit/pit6.png)

We can add with `sudo nano /etc/hosts` the recently discovered host to our `/etc/hosts` file, so we can connect to it.

![](/assets/imageshtb-writeup-pit/pit7.png)

## Nmap - UDP

As this information is not enough, as we don't have any vector to follow, we can scan the main UDP ports with `nmap -sU -top-ports <n> pit.htb`, and when we discover some relevant ports as 135, 161 or 162, we can scan them with the `-sV` parameter to reveal the service and version running there.

We can see that SNMP is listening on the default port.

![](/assets/imageshtb-writeup-pit/pit2.png)

## Nginx

The first web server, in the port 80, contains a default Nginx welcome page. There is nothing to do here, for now.

![](/assets/imageshtb-writeup-pit/pit3.png)

With the dms-pit.htb host, we get a 403 forbidden error.

![](/assets/imageshtb-writeup-pit/pit8.png)

## CentOS

In the another port, there is a login pannel of CentOS Linux. As the default credentials are not working, there is also nothing to do in this page.

![](/assets/imageshtb-writeup-pit/pit4.png)

## SNMP

To enumerate SNMP, we could use `snmpwalk`, but I prefered an interesting Perl script named `snmpbw.pl` because it is really easy to use.

![](/assets/imageshtb-writeup-pit/pit5.png)

If it sends an error, you can use `sudo cpan -i NetAddr::IP` to fix it.

![](/assets/imageshtb-writeup-pit/pit9.png)

With the command working propperly, we can access using the default `public` community. It will generate a .snmp file that we can read.

![](/assets/imageshtb-writeup-pit/pit11.png)

This is a really long file with a lot of useful information. We have to look carefully to it so we can find important paths, users, and relevant info.

![](/assets/imageshtb-writeup-pit/pit13.png)

We first see a relevant path, `/var/www/html/seeddms51x/seeddms`. This seems to be mounted under the web root.

![](/assets/imageshtb-writeup-pit/pit14.png)

Later, at the end of the file we can find a username, `michelle`.

![](/assets/imageshtb-writeup-pit/pit16.png)

## Foothold

As we know the directory `/seedms51x/seeddms`, we can access there with the dms-pit.htb URL.

![](/assets/imageshtb-writeup-pit/pit15.png)

With the credentials `michelle:michelle` (Random testing), I managed to get into the SeedDMS system.

![](/assets/imageshtb-writeup-pit/pit17.png)

There, we can upload files, so we can just upload a PHP Reverse Shell.

![](/assets/imageshtb-writeup-pit/pit18.png)

This procedure came from an available RCE exploit (CVE-2019-12744). It works as shown below:

![](/assets/imageshtb-writeup-pit/pit20.png)

So, with our shell uploaded and navigating to the propper path, we can seet the cmd variable to the command we want to execute. In this case, I dropped the `conf/settings.xml`, which contains database credentials.

![](/assets/imageshtb-writeup-pit/pit21.png)

I saved the credentials in my workspace to use them later.

![](/assets/imageshtb-writeup-pit/pit23.png)

## CentOS exploitation

With those credentials, I tried to access to the CentOS login page.

![](/assets/imageshtb-writeup-pit/pit24.png)

It didn't work at first, but it did with the user `michelle` instead of `seeddms`.

![](/assets/imageshtb-writeup-pit/pit25.png)

Here, we can see a control pannel of what seems to be a virtual machine.

![](/assets/imageshtb-writeup-pit/pit26.png)

The interesting point is that we can run any command in an interactive terminal available in the web.

![](/assets/imageshtb-writeup-pit/pit27.png)

As user michelle, we can now read the user flag.

![](/assets/imageshtb-writeup-pit/pit28.png)

## Privilege Escalation

In the available binaries for this user, we can see one named `monitor`, which doesn't sound familiar.

![](/assets/imageshtb-writeup-pit/pit29.png)

If we read this binary, we will see a simple bash script.

![](/assets/imageshtb-writeup-pit/pit30.png)

To exploit this, we can start generating rsa keys.

![](/assets/imageshtb-writeup-pit/pit31.png)

I wrote a command that will copy my key into the root authorized ssh keys, if executed as root.

![](/assets/imageshtb-writeup-pit/pit32.png)

From the target machine, I downloaded this script as `exploit.sh`

![](/assets/imageshtb-writeup-pit/pit33.png)

Using `snmpwalk` we can execute the script. The explanation is that we have write access to the monitoring directory. The difficult point is that a TCP connection won't work, as we saw before in this machine (SELinux rules).

![](/assets/imageshtb-writeup-pit/pit36.png)

If it worked propperly, we can now connect with our keys as root to the machine, and get the root flag.

![](/assets/imageshtb-writeup-pit/pit37.png)