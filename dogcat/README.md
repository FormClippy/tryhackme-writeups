## ✨ บทนำ
ห้อง Dogcat เป็นห้อง CTF ที่เน้นการโจมตีผ่านช่องโหว่ Local File Inclusion (LFI) และการยกระดับสิทธิ์ในระบบ โดยมีเป้าหมายหลักคือการเจาะระบบและอ่านไฟล์ flag ที่ซ่อนอยู่

## 🔍 เป้าหมายของโจทย์
- **เจาะระบบเว็บเซิร์ฟเวอร์**
  - ใช้ช่องโหว่ Local File Inclusion (LFI) เพื่อเข้าถึงระบบ
  - ยกระดับสิทธิ์จากผู้ใช้เว็บ (`www-data`) เป็นผู้ดูแลระบบ (`root`)
- **ค้นหาและอ่านไฟล์ Flag ทั้ง 4 ใบ**

# 🧠 TryHackMe - Revenge

> 🟡 หมวด: Web Security / Privilege Escalation  
> 🧩 ความยาก: Medium  
> 🕵️‍♂️ โหมด: Capture The Flag  
> 🔗 URL: [Dogcat](https://tryhackme.com/room/dogcat)
> 👨‍💻 ผู้ทำ: Thanyakorn

---

## 🛰️ 1. ข้อมูลเบื้องต้น (Target Info)
- IP เครื่องเป้าหมาย: `10.10.85.123`
- Web servers: Apache Web Server

## 🚀 ขั้นตอนการโจมมตี
### 🕵️‍♂️ 1. สำรวจช่องโหว่ LFI ผ่านพารามิเตอร์ view
