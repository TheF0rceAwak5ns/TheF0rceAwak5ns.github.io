
---
title: ShaktiCTF 2024- Writeup
date: 2024-03-09 10:11:53 +/-0800
categories: [writeup, CTF]
writeup: :title
permalink: /shaktictf
tags: [htb, writeup, gccctf,web ,talace, 4nh4ck1ne, reverse, web, forensics, crypto]     # TAG names should always be lowercase
---

## Forensics 
### Aqua Gaze
Flag by : **Anakin** & **F4doli**

To start the chanlleges we have a pictures Directly I make a binwalk to extract potential file and hidden directory. 

```sql
> binwalk -e sea.jpeg    

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
512851        0x7D353         Zip archive data, at least v1.0 to extract, name: artofeye/
512918        0x7D396         Zip archive data, encrypted at least v2.0 to extract, compressed size: 299949, uncompressed size: 300029, name: artofeye/artofeye.jpg
813132        0xC684C         End of Zip archive, footer length: 22

```

The extraction have work we get a zip file lets try to extract the data on. 

```sql
>  unzip 7D353.zip           
Archive:  7D353.zip
[7D353.zip] artofeye/artofeye.jpg password: 
```

The archive is protected by a Password lets use zip2john to exract some hash and crack it. 

```sql
> zip2john 7D353.zip > forjohn.txt && john --wordlist=/usr/share/wordlists/rockyou.txt forjohn.txt
```

So the password is `angel1`

Get the second images name `artofeye.jpg` . After lot of try i found a tool checklist made a french guy. 
[https://k-lfa.info/tools-stegano/](https://k-lfa.info/tools-stegano/)

I found a tool which extracts from the hidden data in JPEG files what we have before us, lets try that. 

[GitHub - lukechampine/jsteg: JPEG steganography](https://github.com/lukechampine/jsteg)

```sql
# Download the binary 
> wget https://github.com/lukechampine/jsteg/releases/download/v0.3.0/jsteg-linux-amd64 && chmod +x jsteg-linux-amd64

# Extract data from the JPEG file 
./jsteg-linux-amd64 reveal artofeye/artofeye.jpg > flag.txt
```

It seems we have a large output text file try to analyze it. It seems we have a large output text file try to analyze it. by looking well we can directly fall on a chain of carateres encoded in base64, decode it and gain the flag !! 🤗

## Crypto
### Flag Expedition
Flag by : **Anakin**

The chall begin with this image : 

![intro](assets/ShaktiCTF/flag_expedition/intro.png)

After a fast search on google I find that the flags correspond with a language used in the maritime environment 

[https://fr.wikipedia.org/wiki/Code_international_des_signaux_maritimes#Signaux_de_trois_flottants_commençant_par_M_(Mike)](https://fr.wikipedia.org/wiki/Code_international_des_signaux_maritimes#Signaux_de_trois_flottants_commen%C3%A7ant_par_M_(Mike))

A flag corresponds to a letter as we can see 

![drapeaux](assets/ShaktiCTF/flag_expedition/drapeaux.png)

After interpretation I get the following word ( I put _ to say that it misses these characters) 

`wasittooe__ytof_nd`

With more space to better understanding : 

`was it too e__y to f_nd`

I think we can easily guess the flag 😼

Final flag : `shaktictf{was_it_too_easy_to_find}`

## Web 
### Delicious 
Flag by : **Anakin**

The chall begin on a big cookie screen. 

![intro](assets/ShaktiCTF/Delicious/intro.png)

I do not know you but when I see a cookie I think directly to the session cookie so I look if there is one configured

![cookie](assets/ShaktiCTF/Delicious/fond_cookie.png)

We have a cookie and has seen eye it is encoded in b64, I will use burpsuite to better interact with the data 

When I decode the cookie in burp I can see the admin value is 0

![decode](assets/ShaktiCTF/Delicious/decode_cookie.png)

I can juste change the value to 1 and send the request.

![flag](assets/ShaktiCTF/Delicious/flag.png)

GG we get the Flag ! =) 

### Ultimate Spiderman Fan

Flag by : **Anakin** & **Talace**

We get this page : 

![intro](assets/ShaktiCTF/Ultimate_sipderman/intro.png)

We can see directly that we will have to find a way to buy the **$5,000 Spider Surprize** 

We can buy an item and after that we can do a kind of check to recover a token as it is said in the message 

```texte 

Your Spider Token has been set

Proceed to /checkout to confirm your payment.

```

If we click on checkout it says that the payment is confirmed but if we look at our cookie side we can see that there is a jwt session token that was setup. 

![find_jwt](assets/ShaktiCTF/Ultimate_sipderman/find_jwt.png)

Lets analyse it with burp and is extension **jwt editor**. 

Mainly the jwt is composing the following in decoded form 

```bash
# Header
{"alg": "HS256", "typ": "JWT"}
# Payload
{"amount": 1000}
```

It will therefore be necessary to find the way to resign the token by modifying the payload and its parameter amount = 5000.

Still with burpsuite use the attack `Signe with Psychic Signature` This attaque will update the algorythme of the jwt and if I jwt is vulnerable we will be able to resign it with the new payload 

The new jwt before the attack : 

```bash
# Header
{"typ": "JWT","alg": "ES256"}
# Payload
{"amount": 5000}
```

Now send the request always in `/checkout` and get the flag 😁
![fmag](assets/ShaktiCTF/Ultimate_sipderman/flag.png)
