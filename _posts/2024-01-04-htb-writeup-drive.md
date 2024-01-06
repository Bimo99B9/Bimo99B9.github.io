---
layout: single
title: Drive - Hack The Box
excerpt: "The Drive machine is a hard Linux system that needs reverse engineering, and performing a SQL injection on a binary."
date: 2024-01-04
classes: wide
header:
  teaser: /assets/images/htb-writeup-drive/drive_logo.jpg
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

![](/assets/images/htb-writeup-drive/drive_logo.jpg)

The Drive machine is a hard Linux system that needs reverse engineering, and performing a SQL injection on a binary.

## Web Enumeration

The website is a platform similar to Google Drive, where it is possible to register and upload files.

![](/assets/images/htb-writeup-drive/drive2.png)

The files are obtained by a request to `/id/getFileDetail/`. To enumerate possible files, it is possible to use Burpsuite in Intruder mode by parameterising the id of the file. In this way, depending on the server response (Not Found vs. Unauthorised), it is possible to identify files of other users.

![](/assets/images/htb-writeup-drive/drive1.png)

One of the ids found is `79`. When entering `drive.htb/79/`, you get a Not Found message.

![](/assets/images/htb-writeup-drive/drive4.png)

## Blocking Functionality

The platform has a file "reservation" system, which allows to make files accessible to external users. Below we try to reserve a sample file to see how the web page behaves.

![](/assets/images/htb-writeup-drive/drive5.png)

The file is successfully backed up.

![](/assets/images/htb-writeup-drive/drive6.png)

Using this file as an example, the `/id/block` path is discovered which allows access to a reserved file. Thus, via the path `/79/block/` one of the previously found files can be accessed.

![](/assets/images/htb-writeup-drive/drive7.png)

This file contains a message to the development team with the credentials of "martin".

![](/assets/images/htb-writeup-drive/drive8.png)

These credentials give SSH access with the user martin.

![](/assets/images/htb-writeup-drive/drive9.png)

After enumerating the system, in `var/www/backups` there are a number of backups encrypted in `.7z`, as well as a `db.sqlite3` database.

![](/assets/images/htb-writeup-drive/drive10.png)

With a server up with `python3 -m http.server 8000`, the database is downloaded to our system.

![](/assets/images/htb-writeup-drive/drive12.png)

With `sqlite3` we load the database and find a series of hashes.

![](/assets/images/htb-writeup-drive/drive13.png)

Using `hashid`, we find the mode of the hash.

![](/assets/images/htb-writeup-drive/drive15.png)

However, I was unable to crack these hashes, so I continued to enumerate the machine. It was probably a rabbit hole.

![](/assets/images/htb-writeup-drive/drive17.png)

In the `/usr/local/bin` directory is a `gitea` binary for which you have execute permissions.

![](/assets/images/htb-writeup-drive/drive18.png)

When running it, an error appears as the gitea server is already running, probably because another user already launched it.

![](/assets/images/htb-writeup-drive/drive19.png)


## Gitea

Gitea is accessed by port forwarding with the command `ssh martin@10.10.11.235 -L 3000:drive.htb:3000`.

![](/assets/images/htb-writeup-drive/drive24.png)

This way, in `localhost:3000` we find the Gitea platform.

![](/assets/images/htb-writeup-drive/drive25.png)

Listing the platform, we find the users. One of them corresponds to the user `martin`, for which we have the SSH credentials.

![](/assets/images/htb-writeup-drive/drive26.png)

With the SSH password of `martin` and the user `martinCruz` we get access. We found a repository of the `DoodleGrive` platform.

![](/assets/images/htb-writeup-drive/drive27.png)

In the repository, there is a `db_backup.sh` file containing the credential that encrypts the backups we found earlier.

![](/assets/images/htb-writeup-drive/drive28.png)

We then reuse the open http server with python to download these backups.

![](/assets/images/htb-writeup-drive/drive29.png)

And unzip with the password found.

![](/assets/images/htb-writeup-drive/drive30.png)

Each backup contains a database with different passwords each time, for a number of users. One of them, the most recent, has the credentials hashed with PBKDF2, so we ignore them for now.

![](/assets/images/htb-writeup-drive/drive31.png)

The others use SHA1.

![](/assets/images/htb-writeup-drive/drive32.png)

I gathered all the hashes and started a brute force attack with `hashcat -m 124 -a 0 -O hashes.txt rockyou.txt`.

![](/assets/images/htb-writeup-drive/drive33.png)

Three similar passwords are found with this attack.

![](/assets/images/htb-writeup-drive/drive34.png)

Trying SSH with the other user on the system, `tom`, all three credentials, we find that `johnmayer7` works and we get access.

![](/assets/images/htb-writeup-drive/drive36.png)

## Privilege escalation

A `README.txt` message is found in the system indicating the development of a new project, "DoodleGrive self hosted", a CLI tool that is still under development.

![](/assets/images/htb-writeup-drive/drive39.png)

We transfer the binary to our system and check that when we run it, it asks for some credentials.

![](/assets/images/htb-writeup-drive/drive40.png)

I used `ghidra` to scan the binary for any clues.

![](/assets/images/htb-writeup-drive/drive43.png)

So, with the binary decompiled, in `main` are the credentials that the binary checks for when it asks the user for them.

![](/assets/images/htb-writeup-drive/drive45.png)

When you use them and check that they work, you get six options related to accounts and the server.

![](/assets/images/htb-writeup-drive/drive46.png)

One of them, number 5, attempts to activate the account of a given user. This allows it to receive a username and performs an action on a supposed database.

![](/assets/images/htb-writeup-drive/drive47.png)

Digging into the binary, we find the query that the binary uses to perform this action.

![](/assets/images/htb-writeup-drive/drive48.png)

You can see that the given user is a parameter of this query that updates the database, which leads us to think that a SQL injection can be performed in the database through the binary from the original system.

![](/assets/images/htb-writeup-drive/drive50.png)

The query is `/usr/bin/sqlite3 /var/www/DoodleGrive/db.sqlite3 -line 'UPDATE accounts_customuser SET is_active=1 WHERE username="%s";'`.

To execute a Remote Code Execution (RCE) attack, the `load_extension()` function of SQLite3 is used.

The binary escapes certain characters, so the `char()` function must be used to send the compiled text to be executed in ASCII.

The payload will be `+load_extension(char(46,47,122))+` where 46 is ".", 47 is "/" and 122 is "z", so you can execute "./z".

![](/assets/images/htb-writeup-drive/drive53.png)

The `z.c` file contains the code that will copy the flag, although you can also run a command to create a reverse shell.

![](/assets/images/htb-writeup-drive/drive51.png)

We then run the payload.

![](/assets/images/htb-writeup-drive/drive54.png)

And we can check that the flag appears in the intended path, `/tmp/ab.txt`.

![](/assets/images/htb-writeup-drive/drive55.png)

With this we finish the Drive machine, very complete and complex at the same time.

![](/assets/images/htb-writeup-drive/drive57.png)