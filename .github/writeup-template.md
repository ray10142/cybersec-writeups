# [Machine/Room Name] — [Platform] — [Difficulty]

> **Date:** YYYY-MM-DD  
> **Platform:** TryHackMe / HackTheBox  
> **Difficulty:** Easy / Medium / Hard  
> **Tags:** `#linux` `#web` `#privesc` `#ctf`

---

## Summary

> One or two sentences. What is this machine about? What is the main attack vector?

---

## Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN nmap/initial.txt <TARGET_IP>
```

**Open ports:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH x.x |
| 80   | HTTP    | Apache x.x |

### Web enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Interesting findings:**
- 

---

## Exploitation

### Vulnerability found

> Describe what you found and why it is vulnerable.

```bash
# Command or payload used
```

**Result:** Got a shell as `www-data`

---

## Privilege Escalation

### Enumeration

```bash
sudo -l
find / -perm -4000 2>/dev/null
```

### Method used

> Explain the privesc vector.

**Reference:** [GTFOBins](https://gtfobins.github.io/)

```bash
# Command used to escalate
```

---

## Flags

| Flag | Value |
|------|-------|
| User | `THM{xxxx}` |
| Root | `THM{xxxx}` |

---

## Lessons Learned

- 
- 

---

## References

- [HackTricks](https://book.hacktricks.xyz/)
- [GTFOBins](https://gtfobins.github.io/)
