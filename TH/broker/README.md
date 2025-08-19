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
- ⚡ `-sS` → SYN Scan (Half-open) เร็วและ stealthy กว่าการเชื่อมต่อเต็มรูปแบบ
- 🎯 `-p 1001-9999` → กำหนดช่วงพอร์ต 1001–9999
- 🚀 `-T5` → ใช้ความเร็วสูงสุด (Aggressive timing) → สแกนไวขึ้น แต่มีโอกาสถูกตรวจจับได้
- 🌐 `10.10.94.190` → IP ของเครื่องเป้าหมาย

📊 ผลลัพธ์การสแกน
พบว่าพอร์ตที่เปิดอยู่คือ: 
| Port      | Service                       |
|-----------|-------------------------------|
| 1883/tcp | MQTT                          |
| 8161/tcp | Apache ActiveMQ Web Console   |

> 👉 จากขั้นตอนนี้ เราทราบแล้วว่าเครื่องเป้าหมายรัน MQTT บนพอร์ต 1883 และ ActiveMQ Web Console บนพอร์ต 8161 ซึ่งจะถูกใช้ในการโจมตีขั้นตอนต่อไป

## 🌐 ขั้นตอนที่ 2 : วิเคราะห์บริการ (Service Enumeration)

📌 โจทย์:
What is the name of the software they use? (ซอฟต์แวร์ที่เครื่องเป้าหมายใช้งานคืออะไร?)

🔍 การทดสอบบริการที่พบ

![test](images/2.png)

- เมื่อทดลองเข้าที่พอร์ต 1883 → ไม่สามารถเข้าผ่านเว็บเบราเซอร์ได้ เนื่องจากพอร์ตนี้ใช้สำหรับ MQTT (Message Queuing Telemetry Transport)
  - โปรโตคอลนี้มักถูกใช้ในงาน IoT (Internet of Things)
  - ไม่ใช่โปรโตคอลที่รองรับการแสดงผลผ่านเว็บโดยตรง
 
![test](images/3.png)

- เมื่อทดลองเข้าที่พอร์ต 8161 → พบหน้า Web Console ที่สามารถเข้าผ่านเบราเซอร์ได้
  - พอร์ตนี้เป็นพอร์ตมาตรฐานของ Apache ActiveMQ
  - ActiveMQ คือ Message Broker ที่ใช้จัดการ message queuing
  - มีหน้าที่ส่งและรับข้อความระหว่างแอปพลิเคชันต่าง ๆ
  - รองรับการทำงานแบบ Asynchronous Communication (ไม่ต้องรอผลลัพธ์ทันที)
  - มักถูกใช้เพื่อเชื่อมโยงและประสานงานระหว่างระบบที่แตกต่างกัน
 
> 👉 จากจุดนี้ เราทราบแล้วว่าระบบรัน ActiveMQ Web Console อยู่ที่พอร์ต 8161 ซึ่งจะเป็นกุญแจสำคัญในการเจาะระบบขั้นตอนต่อไป

## 🔐 ขั้นตอนที่ 3 : เข้าสู่ระบบ ActiveMQ

📌 โจทย์:
Which videogame are Paul and Max talking about? (พอลกับแม็กซ์กำลังพูดถึงวิดีโอเกมอะไรอยู่?)

🖥️ การเข้าถึง Web Console
- เมื่อเข้าไปที่ พอร์ต 8161 จะเห็นหน้าเว็บ Apache ActiveMQ Console
- ลองเข้าสู่ระบบด้วย credential ค่าเริ่มต้น:
  - admin/password → ❌ ใช้งานไม่ได้
  - admin/admin → ✅ สามารถล็อกอินได้สำเร็จ

![login](images/4.png)

🔎 ข้อมูลที่ค้นพบ
- หลังจากล็อกอิน พบว่า ActiveMQ Version ที่รันอยู่คือ:
  
  🧩 Apache ActiveMQ 5.9.0

> 👉 จากจุดนี้ เราสามารถเข้าถึงเมนูจัดการของ ActiveMQ ได้แล้ว และจะนำไปสู่การค้นหาข้อความที่ Paul และ Max พูดคุยกันในขั้นตอนถัดไป

## 📡 ขั้นตอนที่ 4 : อ่านข้อความลับจาก MQTT

🔎 การตรวจสอบ Topic
- หลังจากล็อกอินเข้าสู่ ActiveMQ Console
- เมื่อกดไปที่หัวข้อ Topics จะพบตารางแสดงรายการต่าง ๆ
- มี Topic ที่น่าสนใจชื่อว่า secret_chat ซึ่งมีข้อความถูก queue อยู่ประมาณ 64 ข้อความ
- นี่น่าจะเป็นแหล่งข้อมูลการสนทนาของ Paul และ Max

💻 การเขียน Client เพื่อ Subscribe ข้อความ
เพื่ออ่านข้อความใน secret_chat เราต้องเขียน MQTT client ขึ้นมา:
1. สร้างไฟล์ชื่อ client.py
2. วางโค้ดด้านล่างนี้ลงไป
