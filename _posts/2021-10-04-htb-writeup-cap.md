---
layout: single
title: Cap - Hack The Box
excerpt: "Cap is one of the easiest Linux machines available in the platform. It is the machine that I always recommend to my degree partners that want to start in HackTheBox, as it is very intuitive and the required tools are known for every person with IT knowledges, even if it is their first machine."
date: 2021-10-04
classes: wide
header:
  teaser: /assets/images/htb-writeup-cap/cap_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - easy
  - infosec
tags:  
  - Wireshark
  - Linpeas
  - SSH
  - GTFOBins
---

![](/assets/images/htb-writeup-cap/cap_logo.jpg)

Cap is one of the easiest Linux machines available in the platform. It is the machine that I always recommend to my degree partners that want to start in HackTheBox, as it is very intuitive and the required tools are known for every person with IT knowledges, even if it is their first machine.

## Nmap

The Nmap scan reveals a web server, and a SSH service in the machine.

![](/assets/images/htb-writeup-cap/cap3.png)

## Dashboard

If you go into the web, you will find a dashboard with some graphs. You can see at the top right the user "Nathan" and some pages in the left column.

![](/assets/images/htb-writeup-cap/cap1.png)

The "security snapshot" page, has statistics about some TCP packets sent and received. There is a download button that will download a .pcap file that, if we open with Wireshark, we won't find anything interesting.

![](/assets/images/htb-writeup-cap/cap4.png)

The point here is that each time you reload the web, the url gets updated with one more number (i++). This generates a new .pcap file with the newest TCP packets. You can deduct from here that the first packets are in the `/data/0` directory. If you go there, the file becomes interesting.

## Wireshark

If you carefully inspect the packets with Wireshark, you will find the credentials of a FTP connection, which are the same for SSH.

![](/assets/images/htb-writeup-cap/cap6.png)

Now we can log in with SSH and get the first flag.

![](/assets/images/htb-writeup-cap/cap7.png)

## Privilege escalation

If we run Linpeas (available in the home folder in the system), you will notice that the CAP_SETUID capability is set by the Python binary. This can be easily exploited to get a bash as the root user. We can find more about this in GTFOBins.

![](/assets/images/htb-writeup-cap/cap9.png)

Now, with a single command, we will upgrade our shell.

![](/assets/images/htb-writeup-cap/cap10.png)

Here ends this machine. As I firstly commented, one of the easiest ones in the HackTheBox platform, but very welcoming for the new users.