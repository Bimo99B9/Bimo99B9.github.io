---
layout: single
title: Dynstr - Hack The Box
excerpt: "Dynstr is a different box that works with dynamic dns and a really uncommon privilege escalation. It is not an OSCP style box, but it is also interesting because of how different it is. We can learn many things about DNS with this system."
date: 2021-10-17
classes: wide
header:
  teaser: /assets/images/htb-writeup-dynstr/dynstr_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - medium
  - infosec
tags:  
  - DNS
  - base64
  - nsupdate
---

![](/assets/images/htb-writeup-dynstr/dynstr_logo.jpg)

Dynstr is a different box that works with dynamic dns and a really uncommon privilege escalation. It is not an OSCP style box, but it is also interesting because of how different it is. We can learn many things about DNS with this system.

## Nmap

First, we can add the main host of the machine to our `/etc/hosts` file.

![](/assets/images/htb-writeup-dynstr/dynstr1.png)

The nmap reports three open ports, 22, 53 and 80.

![](/assets/images/htb-writeup-dynstr/dynstr3.png)

## Website enumeration

The website looks like a Dns site. We can look carefully, and notice about the *dynamic DNS* in the subtitle. We will see this later.

![](/assets/images/htb-writeup-dynstr/dynstr4.png)

The most relevant information we can get in this web, is in the bottom. We can see some domains, so we can say that there is virtual hosting in the machine. We can also save the credentials in the bottom right.

![](/assets/images/htb-writeup-dynstr/dynstr5.png)

We can add the hosts to our `/etc/hosts` file.

![](/assets/images/htb-writeup-dynstr/dynstr6.png)

If we do fuzzing in the main host, we can find an interesting directory, **/nic**.

![](/assets/images/htb-writeup-dynstr/dynstr7.png)

In this directory, thre is another one, **/update**.

![](/assets/images/htb-writeup-dynstr/dynstr8.png)

## Dynamic DNS

In this URL, we can see the text *badauth*.

![](/assets/images/htb-writeup-dynstr/dynstr9.png)

If we investigate about this, we can find a useful no-ip post explaining it. We have to base64 encode the credentials in the request.

![](/assets/images/htb-writeup-dynstr/dynstr10.png)

The request will look like this.

![](/assets/images/htb-writeup-dynstr/dynstr12.png)

And with the base64 encoding:

![](/assets/images/htb-writeup-dynstr/dynstr13.png)

The server response is a 200 OK if everything worked propperly.

![](/assets/images/htb-writeup-dynstr/dynstr14.png)

## Reverse shell

We can play with the base64 encoding. Let's encode a simple bash reverse shell.

![](/assets/images/htb-writeup-dynstr/dynstr15.png)

With this payload, if we listen in our machine with nc in the specified port, we will get a reverse shell as the `www-data` user.

![](/assets/images/htb-writeup-dynstr/dynstr17.png)

## Obtaining a user shell

Enumerating a little bit shows us an interesting file.

![](/assets/images/htb-writeup-dynstr/dynstr18.png)

In that file there is a private key that we can save in our machine.

![](/assets/images/htb-writeup-dynstr/dynstr19.png)

With `chmod 600` we make the file a useful SSH key.

![](/assets/images/htb-writeup-dynstr/dynstr20.png)

Now, the problem is that this key is not working because our host is not a valid one from the victim machine eyes. We have to allocate it in its scope.

I found how to add dns records in the following site.

![](/assets/images/htb-writeup-dynstr/dynstr21.png)

Then, we can do it in the victim machine with `nsupdate`.

![](/assets/images/htb-writeup-dynstr/dynstr22.png)

Now we can use the key and connect ad the `bindmgr` user, and get the user flag.

![](/assets/images/htb-writeup-dynstr/dynstr23.png)

## Privilege escalation

With this user we can run a script as root. We can check that with `sudo -l`.

![](/assets/images/htb-writeup-dynstr/dynstr24.png)

If we read the script, we can see a vulnerable line.

![](/assets/images/htb-writeup-dynstr/dynstr25.png)

We can take profit of this copying the `/bin/bash` to the target directory, and assign some privileges to the bash.

![](/assets/images/htb-writeup-dynstr/dynstr27.png)

We are now root.