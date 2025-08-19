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
> 🧩 URL: Broker
> 👨‍💻 ผู้ทำ: Thanyakorn
