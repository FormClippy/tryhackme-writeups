### ✨ บทนำ
Broker เป็นห้อง CTF ที่เน้นเจาะระบบ Apache ActiveMQ ซึ่งทำงานเป็น Message Broker ผู้เล่นจะต้องสแกนพอร์ต, วิเคราะห์บริการ, exploit ช่องโหว่ (CVE-2016-3088) และยกระดับสิทธิ์จนได้ root

### 🔍 เป้าหมายของโจทย์
- ทำ Port Scan หาพอร์ตที่เปิด
- ระบุ Software ที่ใช้งาน
- อ่านข้อมูลที่ซ่อนใน ActiveMQ (secret_chat)
- Exploit ActiveMQ → Reverse Shell
- ยกระดับสิทธิ์เป็น Root และดึง Flag

🧠 TryHackMe - Broker 📈

> 🟡 หมวด: Web / Privilege Escalation  
> 🧩 ความยาก: Medium  
> 🕵️‍♂️ โหมด: CTF แบบ Capture The Flag  
> 🧩 URL: [Revenge](https://tryhackme.com/room/broker)  
> 👨‍💻 ผู้ทำ: Thanyakorn

## 📚 สารบัญ

- 🛰️ 1. สแกนพอร์ต (Port Scanning)
- 🌐 2. วิเคราะห์บริการ (Service Enumeration)
- 🔐 3. เข้าสู่ระบบ ActiveMQ
- 📡 4. อ่านข้อความลับจาก MQTT
- 🚪 5. Exploit ActiveMQ (CVE-2016-3088)
- 🖥️ 6. Reverse Shell & Initial Access
- 🔼 7. Privilege Escalation → Root
- 🏁 8. เก็บ Flag

---

## 🛰️ ขั้นตอนที่ 1 : การสแกนพอร์ต (Port Scanning)

📌 โจทย์:
ให้ทำการ สแกน TCP Port ทั้งหมดที่มีหมายเลขมากกว่า 1000 และน้อยกว่า 10000 เพื่อระบุว่าพอร์ตใดบ้างที่เปิดใช้งานอยู่บนเครื่องเป้าหมาย

💻 คำสั่งที่ใช้

```bash
nmap -sS -p 1001-9999 -T5 10.10.94.190
```

![nmap](images/1.png)

🧩 อธิบายพารามิเตอร์
- ⚡ -sS → SYN Scan (Half-open) เร็วและ stealthy กว่าการเชื่อมต่อเต็มรูปแบบ
- 🎯 -p 1001-9999 → กำหนดช่วงพอร์ต 1001–9999
- 🚀 -T5 → ใช้ความเร็วสูงสุด (Aggressive timing) → สแกนไวขึ้น แต่มีโอกาสถูกตรวจจับได้
- 🌐 10.10.94.190 → IP ของเครื่องเป้าหมาย
