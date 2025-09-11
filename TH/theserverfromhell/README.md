### ✨ บทนำ
The Server From Hell เป็นห้อง CTF สาย Linux บน TryHackMe ที่เน้นการเจาะระบบตั้งแต่การ enumerate พอร์ตซ่อน, เจอ misconfigured NFS share, crack ไฟล์ zip, ใช้ SSH key เข้าเครื่อง และยกระดับสิทธิ์ด้วย Linux capabilities จนได้ root shell

### 🔍 เป้าหมายของโจทย์
“Gain access to the target machine, escalate your privileges, and capture the root flag.”

สรุป:
- Enumerate พอร์ตที่ถูกซ่อนไว้
- เข้าถึงไฟล์สำคัญจาก NFS share
- Crack password ของ zip เพื่อได้ private key
- ใช้ SSH เข้าเครื่องในฐานะ user hades
- Exploit Linux capability (tar) เพื่อยกระดับสิทธิ์เป็น root
- เก็บ Flags ให้ครบ

# 🧠 TryHackMe - The Server From Hell

🟡 หมวด: CTF / Boot-to-Root  
🧩 ความยาก: Medium
🕵️‍♂️ โหมด: CTF แบบ Boot-to-Root
🧩 URL: The Server From Hell
👨‍💻 ผู้ทำ: Thanyakorn

---

## 📚 สารบัญ
- 📌 ข้อมูลจากโจทย์
- 🛰️ 1. ข้อมูลเบื้องต้น (Target Info)
- 🌐 2. ทดลองเข้าใช้งานบริการ
- 📌 พิจารณาเบื้องต้น
- 🚪 3. Initial Access
- 🔐 4. SSH เข้าเครื่องเป้าหมาย
- 📁 5. ค้นหา Flag
- 🔼 6. Privilege Escalation
- 🏁 7. ค้นหา Root Flag

---

## 📌 ข้อมูลจากโจทย์
> “Start at port 1337 and enumerate your way. Good luck.”

## 🔍 Service Enumeration 

































