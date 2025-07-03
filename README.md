# 🧠 TryHackMe - Revenge

> 🟡 หมวด: Web / Privilege Escalation  
> 🧩 ความยาก: Medium  
> 🕵️‍♂️ โหมด: CTF แบบ Capture Flag
> 🧩 URL: [Revenge](https://tryhackme.com/room/revenge)
> 👨‍💻 ผู้ทำ: Thanyakorn]



---
## ข้อมูลจากโจทย์บอกมาเบื้องต้น

- What I want you to do is simple. Break into the server that's running the website and deface the front page
- มันหมายถึงว่าสิ่งที่ฉันต้องการให้คุณทำนั้นง่ายมากเจาะเข้าไปในเซิร์ฟเวอร์ที่กำลังรันเว็บไซต์และทำให้หน้าแรกเสียหาย


## 🛰️ 1. ข้อมูลเบื้องต้น (Target Info)

- IP เครื่องเป้าหมาย: 10.10.28.30
- พอร์ตที่เปิด: 22, 80
- ลองเข้าใช้งานเว็บไซต์และกดที่เมนู Products
  
![Web site](images/1.png)

> จะเจอรูปสินค้า,ค าอธิบาย,ราคา,สี
> ให้เราสังเกตุ Path ด้านบน จะเป็น /products/1
> ที่นี้ให้กลับไปหน้าก่อนหน้านี้และก็ LEARN MORE ของอีกสินค้าด

---

## 🔍 2. Enumeration

### 🔸 2.1 Nmap Scan

```bash
nmap -sC -sV -oA revenge-scan 10.10.x.x
