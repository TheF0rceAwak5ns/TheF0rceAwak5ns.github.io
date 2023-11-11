# How to write a write up (yes literally 🥲)

## First, create your file in /_post 🐤

![nameFile](/assets/tutoWU/capture_01.png) 

## Name the file like this:
```bash
YYYY-MM-DD-TITLE.MD
```

## Now, put an header to configure your path, the tag etc.. as follow 😼:
```
---
title: Relevant - Medium THM # name your WU = Name - [Level] [Platform]
date: 2023-11-01 12:01:23 +/-0800 # configure the date same as the name of your file
categories: [writeup, tryhackme] # categories, First the parents than the child-category, 2 or 3 MAX !
writeup: :title # don't touch me! 
permalink: /relevant # / and the name
tags: [thm, medium, writeup, windows, command injection, privilege abuse] #TAG names always lowercase (a,b,c,d,e,f...)
---

- OS: name
- Difficulty: level
- Author: your_name

# Scanning
# Enumeration
# Exploit
# Privilege escalation
# Resources (optional)
```

## ⚠️**IF YOU HAVE IMAGES**⚠️
1. create a folder under /assets/
2. name the folder like this: boxPlatform : authorityHTB
3. put your images in it!
4. In your writeup put this and fill it with your info:
```python
![nameFile](/path/to/your/images.png) #GIF, PNG or JPEG
```

## Now fetch, pull, commit AAAAAAAAAAAAAAAND PUSH !!! 🥳