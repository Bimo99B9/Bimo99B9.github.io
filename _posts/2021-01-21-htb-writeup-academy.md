---
layout: single
title: Academy - Hack The Box
excerpt: "Academy is a very complete Linux machine about the new HackTheBox Academy platform, which covers enumeration with directory lists, virtual hosting, Laravel exploitation, a lot of lateral movement and privilege escalation with composer."
date: 2021-01-21
classes: wide
header:
  teaser: /assets/images/htb-writeup-academy/academy-logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - easy
  - infosec
tags:  
  - Linux
  - laravel
  - fuzzing
  - composer
  - burpsuite
---

![](/assets/images/htb-writeup-academy/academy-logo.png)

Academy is a very complete Linux machine about the new HackTheBox Academy platform, which covers enumeration with directory lists, virtual hosting, Laravel exploitation,  a lot of lateral movement and privilege escalation with composer.

## Portscan

![](/assets/images/htb-writeup-academy/academy1.png)

We can add the ip with the standard host to the `/etc/hosts` file.  

![](/assets/images/htb-writeup-academy/academy2.png)

## Academy web

As we enter to the HTTP web of the machine, we see some register/login buttons.

![](/assets/images/htb-writeup-academy/academy3.png)

We don't have credentials, so we register our own account

![](/assets/images/htb-writeup-academy/academy4.png)

We get the HTB academy webpage, but there’s nothing relevant there. It seems that we will need an admin account.

![](/assets/images/htb-writeup-academy/academy5.png)

Using burp suite, we try to register again. In the 15th row we can see the attributes for the
username and password, but also a “roleid” which is set in 0 by default. We can try to change it to 1 and see if there is any change in the created account

![](/assets/images/htb-writeup-academy/academy6.png)

We get a different account but at the first glance nothing changes, so I though about enumerating directories, just in case any of them is only accesible with this new account.

![](/assets/images/htb-writeup-academy/academy7.png)

## Virtual host

In the admin.php page there is another login form. A normal account does not work, but an admin one does.

![](/assets/images/htb-writeup-academy/academy8.png)

There we find a planner with some tasks of the developers. The "pending" one is relevant, as there is a virtual host hostname, `dev-staging-01.academy.htb`.

![](/assets/images/htb-writeup-academy/academy9.png)

To access the host, we must add it to out `/etc/hosts` file.

![](/assets/images/htb-writeup-academy/academy10.png)

When we connect to it with our browser, we can see some posts about the content manager.

![](/assets/images/htb-writeup-academy/academy11.png)

## Laravel intrusion

After some enumeration, we find that Laravel is running and an app_key.

![](/assets/images/htb-writeup-academy/academy12.png)

We will deserialize the PHP Laravel Framework to get RCE in the system. There is a tool in metasploit to do this easily.

![](/assets/images/htb-writeup-academy/academy13.png)

Now we have to set the required options (base64 app_key, virtual host...). When we run the exploit, we get the reverse shell.

![](/assets/images/htb-writeup-academy/academy14.png)

To improve our shell, we have to do:

```
python3 -c 'import pty; pty.spawn("/bin/sh")'
ctrl + z
stty raw echo
fg + enter
reset
```

![](/assets/images/htb-writeup-academy/academy15.png)

## Lateral movement

In the home directory of the current user, `www-data`, there are many composer and config files.

![](/assets/images/htb-writeup-academy/academy16.png)

In one of them, we can see some credentials.

![](/assets/images/htb-writeup-academy/academy17.png)

If we enumerate the users listing the directories in the `/home` folder, we see the following ones:

![](/assets/images/htb-writeup-academy/academy18.png)

Trying the password in those users, we find that it works for user `cry0lit3`.

![](/assets/images/htb-writeup-academy/academy19.png)

We can now get the user.txt flag.

![](/assets/images/htb-writeup-academy/academy20.png)

## From cry0lit3 to mrb3n

The first step I did was improve the shell as I can log in with the credentials `cry0lit3:mySup3rP4s5w0rd !!` to SSH.

![](/assets/images/htb-writeup-academy/academy21.png)

To escalate privileges we can enumerate the system with the linpeas script, so I copied it to the machine.

![](/assets/images/htb-writeup-academy/academy23.png)

The only interesting thing is that we are part of the group adm, which we can also see with the command `id`.

![](/assets/images/htb-writeup-academy/academy24.png)

Enumerating the available files, we find some audit.log logs.

![](/assets/images/htb-writeup-academy/academy25.png)

If we filter in those files by our id, we will see the commands that our user used in the system.

![](/assets/images/htb-writeup-academy/academy27.png)

The user used the `su` command, and we now have the data that he provided in the command.

![](/assets/images/htb-writeup-academy/academy28.png)

So we only have to translate the data from hexadecimal to text, and we have the password of the user he moved to.

![](/assets/images/htb-writeup-academy/academy29.png)

Testing the credentials with the users we had before, we can now move to mrb3n.

![](/assets/images/htb-writeup-academy/academy30.png)

## Privilege escalation

This new user can run `composer` as root, so we may exploit that.

![](/assets/images/htb-writeup-academy/academy31.png)

At first, we will generate a pair of RSA keys in our local machine.

![](/assets/images/htb-writeup-academy/academy32.png)

Then, we write a .json script to execute with composer. When executed, this script will include the generated id_rsa to the authorized_keys of the victim machine.

![](/assets/images/htb-writeup-academy/academy34.png)
![](/assets/images/htb-writeup-academy/academy33.png)

Now, using `sudo composer run-script command` we will perform the keys inclusion.

![](/assets/images/htb-writeup-academy/academy35.png)

At the end, we can finally use the ssh key to connect with ssh to the machine.

![](/assets/images/htb-writeup-academy/academy36.png)

The `root.txt` flag is as always in de `/root` directory, and it is possible to get it now.