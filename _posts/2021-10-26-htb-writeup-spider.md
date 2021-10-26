---
layout: single
title: Spider - Hack The Box
excerpt: "Spider is a complex machine with two SSTI vulnerabilities, and a really interesting method to get cookies with its private key. To escalate privileges we take advantage of the fact that we are allowed to enter input in an XML file, of which a parameter is displayed in a web service."
date: 2021-10-26
classes: wide
header:
  teaser: /assets/images/htb-writeup-spider/spider_logo.jpg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - hard
  - infosec
tags:  
  - SSTI
  - Cookie hijacking
  - SQL
  - XXE
---

![](/assets/images/htb-writeup-spider/spider_logo.jpg)

Spider is a complex machine with two SSTI vulnerabilities, and a really interesting method to get cookies with its private key. To escalate privileges we take advantage of the fact that we are allowed to enter input in an XML file, of which a parameter is displayed in a web service.

## Nmap

The first we do is add the main host of the machine to our `/etc/hosts` file.

![](/assets/images/htb-writeup-spider/spider1.png)

There are only 2 open ports in the machine.

![](/assets/images/htb-writeup-spider/spider2.png)

Here, we can see that the port 80 hosts a nginx web server. The machine is a Linux OS and has Secure Shell Connection enabled.

![](/assets/images/htb-writeup-spider/spider3.png)

With whatweb we can see that same information about the web server.

![](/assets/images/htb-writeup-spider/spider4.png)

## Web page

The page seems to be a furniture shop. We can look carefully and see the login, admin and register pages in the left.

![](/assets/images/htb-writeup-spider/spider5.png)

To login we are asked for an UUID and a password. We will have to register first.

![](/assets/images/htb-writeup-spider/spider6.png)

I checked for SQL injections in there parameters but they are not vulnerable.

![](/assets/images/htb-writeup-spider/spider7.png)

Then, we proceed to register. We need a username and a password.

![](/assets/images/htb-writeup-spider/spider8.png)

When we click the register button, we are given a UUID. With this UUID we can try to login.

![](/assets/images/htb-writeup-spider/spider9.png)

## SSTI

As nothing happened, I thought about a SSTI, a vulnerability I have seen in a recent machine. A SSTI (Server side template injection) plays with the response of the server to execute what we want, as we will see.

To check if it was vulnerable, I registered with `{{ 9 * 9 }}` as my username. If it was vulnerable, the output of the operation will be shown as the username.

![](/assets/images/htb-writeup-spider/spider11.png)

Now, our username is `81`, so it worked.

![](/assets/images/htb-writeup-spider/spider12.png)

We can try to run `{{config}}` to get configuration information.

![](/assets/images/htb-writeup-spider/spider13.png)

Now, in our username there is a lot of information about the web server.

![](/assets/images/htb-writeup-spider/spider15.png)

If we read all the string, we will see a key. This "SECRET_KEY" is used for cookie encoding. With this, we will be able to get the session cookie of the user we want.

![](/assets/images/htb-writeup-spider/spider16.png)

## Session cookies with Flask Unsign

It is possible to craft the cookie we want using the tool `Flask Unsign`, available in Github.

![](/assets/images/htb-writeup-spider/spider18.png)

With `flask-unsign --sign --cookie`, the vulnerable field and the key, we will get a cookie.

![](/assets/images/htb-writeup-spider/spider19.png)

If we replace our cookie with this one, our user will change.

![](/assets/images/htb-writeup-spider/spider20.png)

We are `chiv` at this point. As the admin board is empty, we can suppose that there will be a board that is not empty. So we now need to move to another user.

![](/assets/images/htb-writeup-spider/spider21.png)

## SQL injection in the cookie

This phase is the most difficult one of the machine. We can only get to this by trying everything we know. And I thought about a simple Python oneliner to use flask unsign and the key. With this oneliner, I could test for SQL injections and is shown below.

The "script" is the following one.

```python
from flask_unsign import session as s;
session = s.sign({'uuid': session}, secret = '(SecretKey)')
```

And the SQLmap command to detect SQL injections is:

![](/assets/images/htb-writeup-spider/spider23.png)

Luckily, the parameter Cookie is injectable.

![](/assets/images/htb-writeup-spider/spider24.png)

Now we know chiv's password and uuid.

![](/assets/images/htb-writeup-spider/spider25.png)

We can also see a really interesting message in its dashboard.

![](/assets/images/htb-writeup-spider/spider26.png)

I saved the information retrieved in my content folder.

![](/assets/images/htb-writeup-spider/spider27.png)

With the credentials, we can login.

![](/assets/images/htb-writeup-spider/spider28.png)

And in the dashboard, there is the message we have seen before.

![](/assets/images/htb-writeup-spider/spider29.png)

If we navigate to that URL, we will see a support portal.

![](/assets/images/htb-writeup-spider/spider30.png)

## From SSTI to RCE

This part is also hard, because we have to get a reverse shell from the SSTI vulnerability available in this kind of forms. The full explanation can be read in the following image.

![](/assets/images/htb-writeup-spider/spider31.png)

In internet there are many tutorials to get command execution with SSTI. The problem is that the page has *a lot* of filters, so we can not use *many* restricted characters and symbols. So we need to bypass them, and I found this guide.

![](/assets/images/htb-writeup-spider/spider32.png)

Here, we can see that the curly brackets are forbidden... Then, we have to bypass that restriction.

![](/assets/images/htb-writeup-spider/spider33.png)

In this first bypass, we can replace the inner curly brackets of **"{{ }}"** with **"% %"**

![](/assets/images/htb-writeup-spider/spider34.png)

But again, we are told that **"."**, the point, is a forbidden character.

![](/assets/images/htb-writeup-spider/spider35.png)

We can encode the whole payload with `echo -n (base64) | base64 -d | bash` and pass our payload encoded to bypass the restrictions.

![](/assets/images/htb-writeup-spider/spider36.png)

Our bash payload will be this one.

![](/assets/images/htb-writeup-spider/spider37.png)

And the final payload, with the encoding thechnique, is this one.

![](/assets/images/htb-writeup-spider/spider38.png)

At first, it seems that it does not like it.

![](/assets/images/htb-writeup-spider/spider39.png)

But we finally get a reverse shell in our nc listener!

![](/assets/images/htb-writeup-spider/spider40.png)

## User SSH

We are now **user** in the system.

![](/assets/images/htb-writeup-spider/spider41.png)

To make it easier for us, we will get the *chiv* id_rsa, and use it to connect with SSH.

![](/assets/images/htb-writeup-spider/spider42.png)

We have now a SSH connection as the chiv user.

![](/assets/images/htb-writeup-spider/spider43.png)

## System enumeration

With `netstat -tunlp` we can see the listening ports in the victim machine. The 8080 one seems interesting, as it is normally associated with HTTPS services.

![](/assets/images/htb-writeup-spider/spider44.png)

We can port forward it with SSH so we can get access to the port from our local machine.

![](/assets/images/htb-writeup-spider/spider45.png)

In the webpage hosted in that port there is a "Beta login" form.

![](/assets/images/htb-writeup-spider/spider46.png)

When we enter a username, a similar page as the first one is shown. And more important, our input is reflected.

![](/assets/images/htb-writeup-spider/spider47.png)

## XXE

If we intercept the POST request with Burpsuite, we will see a hidden parameter, `version`.

![](/assets/images/htb-writeup-spider/spider48.png)

We can deduce from here that the application is parsing XML documents. This is vulnerable to XXE (External XML Entity).

![](/assets/images/htb-writeup-spider/spider49.png)

Let's see if we can get something else about the XML document fetched. We can retrieve our session cookie from the logout request.

![](/assets/images/htb-writeup-spider/spider51.png)

If we decode this cookie, and then the base64, we will see a XML syntax with our username and the provided version. We know that the `username` part is reflected in the web, and we can write in the `1.0.0` part. We can play with that.

![](/assets/images/htb-writeup-spider/spider52.png)

Our XML file is the following one:

```XML
<!-- API Version 1.0.0 -->
<root>
  <data>
    <username>bimo</username>
    <is_admin>0</is_admin>
  </date>
</root>
```

Therefore, we can close the API Version comment from our input, and then inject a malicious entity to read whatever we want. And, from our username, we can make a reference to that entity with a pointer.

The payload for the version input parameter will be:

![](/assets/images/htb-writeup-spider/spider53.png)

With this, our XML will look as:

```XML
<!-- API Version 1.0.0 -->
<!DOCTYPE foo> [
  <!ELEMENT foo ANY>
  <!ENTITY bimo SYSTEM "file:///etc/passwd" >
]> <!-- -->
<root>
  <data>
    <username>&bimo;</username>
    <is_admin>0</is_admin>
  </date>
</root>
```

I injected it using burpsuite, because of the URL encoding.

![](/assets/images/htb-writeup-spider/spider55.png)

If everything worked as intended, we will see the file in the webpage.

![](/assets/images/htb-writeup-spider/spider60.png)

With `CTRL+U` we can see read it better.

![](/assets/images/htb-writeup-spider/spider61.png)

Now, we can use this vulnerability to read every file we want. For example, the RSA Private key of the root user (`/root/.ssh/id_rsa`).

![](/assets/images/htb-writeup-spider/spider63.png)

Using this private key, we can connect with SSH as the privileged user of the victim machine.

![](/assets/images/htb-writeup-spider/spider64.png)