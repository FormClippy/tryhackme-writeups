### ✨ บทนำ
The Server From Hell เป็นห้อง CTF สาย Linux บน TryHackMe ที่เน้นการเจาะระบบตั้งแต่การ enumerate พอร์ตซ่อน, เจอ misconfigured NFS share, crack ไฟล์ zip, ใช้ SSH key เข้าเครื่อง และยกระดับสิทธิ์ด้วย Linux capabilities จนได้ root shell

## 🎯 เป้าหมาย
- หาพอร์ต/บริการที่ “ซ่อน” ไว้และยืนยันพฤติกรรมด้วย `nc`
- ใช้ **NFS (2049/tcp)** ที่ misconfigured เพื่อ “ดึงกุญแจ SSH” ของผู้ใช้
- หลบออกจาก **IRB (Interactive Ruby)** ไปเป็นเชลล์จริง
- ใช้ **capabilities ของ /bin/tar** เพื่ออ่านไฟล์ต้องห้าม → ได้ hash ของ root
- Crack hash จนได้รหัสผ่าน → `su root` และเก็บ flag

# 🧠 TryHackMe - The Server From Hell

🟡 หมวด: CTF / Boot-to-Root  
🧩 ความยาก: Medium
🕵️‍♂️ โหมด: CTF แบบ Boot-to-Root
🧩 URL: The Server From Hell
👨‍💻 ผู้ทำ: Thanyakorn

---

## 📚 สารบัญ
- 🔎 1) Recon & Port Scan
- 🧵 2) Banner/Manual Check ด้วย `nc`
- 📦 3) NFS Enumeration & Mount
- 🔑 4) SSH ด้วยกุญแจที่ดึงมา
- 🧯 5) หลบออกจาก IRB → ได้ Bash จริง
- 🧗 6) Privilege Escalation ด้วย Linux Capabilities (tar)
- 🧨 7) Crack `/etc/shadow` → `su root`
- 🏁 8) สรุปเวิร์กโฟลว์ + บทเรียน

---

## 📌 ข้อมูลจากโจทย์
> “Start at port 1337 and enumerate your way. Good luck.”

## 🔍 Recon & Port Scan
เริ่มจากลองใช้คำสั่ง:
```bash
nmap -sC -sV -Pn <TARGET_IP>
```
- `-sC` → รัน Nmap Default Scripts (NSE) เพื่อหาข้อมูลลึก เช่น SSL cert, FTP anon, SSH key fingerprint
- `-sV` → ตรวจสอบเวอร์ชัน service
- `-Pn` → การทดสอบ service แม้ไม่รู้ว่า host ออนไลน์หรือไม่

❌ ปัญหา: การใช้ -sC จะทำให้ Nmap เรียกใช้ script เยอะมาก → แต่ละ script ต้องรอ response หรือ timeout จาก service → ทำให้ **สแกนช้า** โดยเฉพาะถ้าเจอพอร์ตเปิดหลายตัว

เพื่อความรวดเร็วใช้คำสั่ง:
```bash
nmap -sV -vv <TARGET_IP>
```
- ตัด `-sC` ออก → ไม่ต้องรัน NSE scripts → scan ไวกว่า
- ใช้ `-sV` เพียงอย่างเดียว → เน้นดู service และ version
- เพิ่ม `-vv` (very verbose) → ให้ Nmap รายงานผลทีละพอร์ตแบบ realtime ทำให้เห็นผลลัพธ์ “เร็วขึ้น” ถึงแม้เบื้องหลังยังสแกนอยู่

✅ แบบนี้เราจะได้ พอร์ตเปิด + service ที่รันอยู่ เร็วพอที่จะนำไปทำ banner grabbing/manual test ต่อ เช่น ใช้ nc



























