# 🧠 TryHackMe - Revenge

> 🟡 หมวด: Web / Privilege Escalation  
> 🧩 ความยาก: Medium  
> 🕵️‍♂️ โหมด: CTF แบบ Capture The Flag  
> 🧩 URL: [Revenge](https://tryhackme.com/room/revenge)  
> 👨‍💻 ผู้ทำ: Thanyakorn

---

## 📌 ข้อมูลจากโจทย์

> "What I want you to do is simple. Break into the server that's running the website and deface the front page."

💬 แปล: สิ่งที่โจทย์ต้องการคือ เจาะเข้าเว็บและแก้หน้าแรก (deface)

---

## 🛰️ 1. ข้อมูลเบื้องต้น (Target Info)

- IP เครื่องเป้าหมาย: 10.10.28.30  
- พอร์ตที่เปิด: 22, 80

---

## 🌐 2. ทดลองเข้าใช้งานเว็บไซต์

- ไปที่เมนู **Products** จะเจอชื่อสินค้า 4 รายการ  
- คลิก **LEARN MORE** ของสินค้าใดก็ได้

![Web site](images/1.png)

- ตัวอย่างสินค้า: Box of Duckies  
- จะแสดงรูปสินค้า, คำอธิบาย, ราคา, สี  
- สังเกต Path ด้านบนจะเป็น `/products/1`  
- กลับไปหน้าเดิม แล้วคลิก **LEARN MORE** ของสินค้าอื่น

![Products-Path](images/2.png)

- จะเห็นว่า Path เหมือนเดิมแต่ตัวเลขเปลี่ยนเป็น `/products/2`

![Products-Path](images/3.png)

- ลองเปลี่ยนตัวเลข 2 เป็น 3 ใน Path เพื่อดูสินค้าชิ้นที่ 3

![Products-Path](images/4.png)

---

## 📌 พิจารณาเบื้องต้น

- น่าจะมีการใช้ฐานข้อมูลเพื่อดึงข้อมูลสินค้าตาม ID ที่ส่งผ่าน URL
- เมื่อลองใส่เครื่องหมาย ' ที่ Path พบว่าเว็บแสดงข้อผิดพลาด HTTP 500 ซึ่งเป็นสัญญาณบ่งชี้ช่องโหว่ SQL Injection

![Products-Path](images/5.png)

---

## 🚪 3 Initial Access

### 🔸 3.1 ตรวจสอบ SQL Injection ด้วย sqlmap

📥 ใช้คำสั่ง:

bash
sqlmap -u "http://10.10.28.30/products/3*" --dbs


![Sqlmap](images/6.png)

📝 **คำอธิบาย:**

- `sqlmap` คือเครื่องมืออัตโนมัติสำหรับตรวจสอบช่องโหว่ SQL Injection และดึงข้อมูลฐานข้อมูล  
- ตัวเลือก `-u` ใช้ระบุ URL ที่ต้องการทดสอบ โดย "http://10.10.28.30/products/3*" ใช้ wildcard 3* เพื่อให้ sqlmap ลองค่าหลาย ๆ ค่าในพารามิเตอร์ id  
- ตัวเลือก `--dbs` สั่งให้ sqlmap ดึงชื่อฐานข้อมูลทั้งหมดที่เชื่อมต่อกับเว็บเป้าหมาย  

![Sqlmap](images/7.png)

💡 ผลลัพธ์: แสดงฐานข้อมูลต่าง ๆ เช่นฐานข้อมูลหลัก duckyinc และฐานข้อมูลระบบของ MySQL เช่น mysql, information_schema เป็นต้น
> เราจะมุ่งเน้นวิเคราะห์ฐานข้อมูล duckyinc ต่อไป เพราะเป็นฐานข้อมูลสำคัญของเว็บนี้

**สรุป:**  
จากการใช้ sqlmap เราพบช่องโหว่ SQL Injection ที่พารามิเตอร์ id ของ URL สามารถเข้าถึงฐานข้อมูลหลักได้ ซึ่งเป็นจุดเริ่มต้นของการเจาะระบบ

### 🔸 3.2 ตรวจสอบโครงสร้างฐานข้อมูล

หลังจากพบว่าฐานข้อมูลหลักคือ duckyinc ขั้นตอนถัดไปคือการตรวจสอบว่าในฐานข้อมูลนี้มีตารางอะไรบ้าง และแต่ละตารางมีคอลัมน์อะไรที่น่าสนใจ

📥 ใช้คำสั่ง:

```bash
sqlmap -u "http://10.10.28.30/products/3*" -D duckyinc --columns
```

![Sqlmap](images/8.png)

📝 **คำอธิบาย:**

- `-D duckyinc` ระบุชื่อฐานข้อมูลที่ต้องการให้ sqlmap ทำการดึงข้อมูล
- `--columns` ให้ sqlmap ดึงชื่อคอลัมน์ทั้งหมดจากทุกตารางภายในฐานข้อมูลที่ระบุ


เมื่อรันคำสั่งแล้ว sqlmap จะถามว่าต้องการดำเนินการต่อหรือไม่ ให้ตอบ Y

![Sqlmap](images/9.png)

💡 ผลลัพธ์: จะเห็นรายชื่อคอลัมน์ทั้งหมดภายในฐานข้อมูล duckyinc ซึ่งเป็นข้อมูลโครงสร้างที่จำเป็นสำหรับการดึงข้อมูลเป้าหมายในขั้นตอนถัดไป

### 🔸 3.3 ดึงข้อมูลจากตาราง user

หลังจากเราทราบโครงสร้างของฐานข้อมูลแล้ว ขั้นตอนนี้จะเป็นการดึงข้อมูลภายในตาราง user ซึ่งอาจมีข้อมูลสำคัญ เช่น รหัสผ่าน หรือค่า Flag ที่เราต้องการ

📥 ใช้คำสั่ง:

```bash
sqlmap -u "http://10.10.28.30/products/3*" -D duckyinc -T user --dump
```

![Sqlmap](images/10.png)

📝 **คำอธิบาย:**

- `-D duckyinc` ระบุชื่อฐานข้อมูลที่ต้องการให้ sqlmap ทำการดึงข้อมูล
- `-T user` ระบุตารางที่ต้องการดึงข้อมูล
- `--dump ให้` sqlmap ดึงข้อมูลทั้งหมดจากตารางที่ระบุ

![Sqlmap](images/11.png)

💡 ผลลัพธ์: สังเกตคอลัมน์ credit_card ของ id = 6 พบ Flag ซ่อนอยู่ เอาไปตอบใน Tryhackme

![flag1](images/12.png)

### 🔸  3.4 ดึงข้อมูลจากตาราง system_user

หลังจากนั้น เราจะโฟกัสที่ตาราง `system_user` ซึ่งคาดว่าอาจมีข้อมูลผู้ใช้งานระบบ เช่น admin หรือข้อมูลที่เกี่ยวข้องกับการยกระดับสิทธิ์

📥 ใช้คำสั่ง:

```bash
sqlmap -u "http://10.10.28.30/products/3*" -D duckyinc --dump -T system_user
```
![Sqlmap](images/13.png)

📝 **คำอธิบาย:**

- `-T system_user` ระบุตารางที่ต้องการ

![Sqlmap](images/14.png)

💡 ผลลัพธ์: พบ username และ password ที่เป็น hash (bcrypt)

### 🔐 3.5 Credential Cracking with John

หลังจากดึงข้อมูลจากตาราง `system_user` เราพบว่า password ของผู้ใช้งานอยู่ในรูปแบบ bcrypt ซึ่งเป็น hash ที่นิยมใช้ในระบบจริง

---

## 🛠️ ขั้นตอนการเตรียมไฟล์ hash

1. สร้างโฟลเดอร์ชื่อ `ducky` แล้วเข้าไปในโฟลเดอร์:

```bash
mkdir ducky
cd ducky
```

2. สร้างไฟล์ชื่อ hashes.txt เพื่อเก็บ hash:

```bash
nano hashes.txt
```

![Crack](images/15.png)

3. วาง bcrypt hash ที่ได้จาก sqlmap (ในที่นี้ใส่เฉพาะของ user server-admin)
> 📌 แนะนำให้ลบ hash อื่น ๆ ออก เพื่อให้ crack ได้เร็วขึ้น

![Crack](images/16.png)

---

4. ตรวจสอบว่า hash ถูกบันทึกเรียบร้อย:

```bash
cat hashes.txt
```

## 🧂 Crack Bcrypt Hash ด้วย John the Ripper
### 📦 เราจะใช้ john คู่กับ rockyou.txt ในการ Dictionary attack:

```bash
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

⚠️ หมายเหตุ:
> ถ้ามันขึ้นว่า No password hashes left to crack หมายความว่า john ได้ทำการถอดรหัสแฮชที่มีอยู่ในไฟล์ hashes.txt เรียบร้อยแล้วหรือไม่มีแฮชที่เหลือให้ทำการถอดรหัสจากไฟล์นั้น

- หลังจากรันเสร็จรอจนกว่าจะ crack สำเร็จ จากนั้นดูผลลัพธ์ที่ได้โดยใช้คำสั่ง:
```bash
john --show hashes.txt
```

![Crack](images/17.png)

💡 ผลลัพธ์ที่ได้:
ได้รหัสผ่าน: inuyasha สำหรับ server-admin

---

## 🔐 4. SSH เข้าเครื่องเป้าหมาย

- หลังจากได้รหัสผ่าน `inuyasha` ซึ่งเป็นของผู้ใช้ `server-admin` เราจะใช้ SSH เพื่อเข้าเครื่องเป้าหมายผ่านพอร์ต 22 ที่เปิดไว้ตั้งแต่ต้น

📥 ใช้คำสั่ง:

```bash
ssh server-admin@10.10.28.30
```

📝 คำอธิบาย:

- SSH (Secure Shell) ใช้ในการเชื่อมต่อแบบปลอดภัยไปยังเครื่องอื่นผ่านเครือข่าย
- เหมาะสำหรับการควบคุมเซิร์ฟเวอร์, จัดการไฟล์, รันคำสั่งระยะไกล
- การเชื่อมต่อผ่าน SSH มีการเข้ารหัสข้อมูลเพื่อความปลอดภัย

เมื่อรันคำสั่ง ระบบจะขอให้กรอกรหัสผ่าน → ให้ใส่ `inuyasha`
\
✅ หากเชื่อมต่อสำเร็จ จะได้ Shell ของเครื่องเป้าหมายในฐานะผู้ใช้ server-admin

![ssh](images/18.png)

---

## 📁 5. ค้นหา Flag

หลังจากเข้าสู่ระบบได้แล้ว ให้ลองดูไฟล์ใน home directory ของ server-admin:

```bash
ls
```

💡 ผลลัพธ์: พบไฟล์ชื่อ flag2.txt

ใช้คำสั่งเพื่ออ่านเนื้อหา:

```bash
cat flag2.txt
```

![flag2](images/19.png)

✅ ได้ Flag ที่สอง นำไปใช้ตอบใน TryHackMe 

![flag2](images/20.png)

---

## 🔼 6. Privilege Escalation

ขั้นตอนต่อไปคือการตรวจสอบสิทธิ์ของผู้ใช้ `server-admin` ว่าสามารถรันคำสั่งในระบบด้วยสิทธิ์ `sudo` ได้หรือไม่

### 🔍 ตรวจสอบสิทธิ์ด้วยคำสั่ง:

```bash
sudo -l
```
![sudo-l](images/21.png)

📝 คำอธิบาย:
- `sudo -l` ใช้ดูว่าเราสามารถใช้สิทธิ์ sudo กับคำสั่งอะไรได้บ้างโดยไม่ต้องใช้รหัสผ่านเพิ่มเติม

💡 ผลลัพธ์: พบว่าเราสามารถใช้ไบนารีของระบบ systemctl เพื่อเริ่มต้น หยุด และรีสตาร์ทบริการ duckyinc ซึ่งเป็นเว็บเซิร์ฟเวอร์ของระบบ

### ✏️ แก้ไขไฟล์ service ด้วย sudoedit

📥 ใช้คำสั่ง:

```bash
sudoedit /etc/systemd/system/duckyinc.service
```

![sudoedit](images/22.png)

💡 เมื่อกด Enter แล้ว ระบบจะเปิดไฟล์ด้วย editor (เช่น nano หรือ vi) ขึ้นมาให้แก้ไข config ได้ทันที

![sudoedit](images/23.png)

🔍  พบว่าแอปพลิเคชันทำงานภายใต้ผู้ใช้ `flask-app` ซึ่งไม่ใช่ `root` 

🔍 สันนิษฐานได้ว่าแอปนี้พัฒนาโดยใช้ **Flask** ซึ่งเป็น Python web framework สำหรับสร้างเว็บไซต์หรือ API

🔍  ต้องทำ Privilege Escalation จากผู้ใช้ `flask-app` ไปเป็น `root`

![sudoedit](images/24.png)

หลังจากแก้ไขไฟล์บริการแล้ว ต้อง reload daemon เพื่อให้ systemd รับการเปลี่ยนแปลง:

```bash
sudo  systemctl daemon-reload
```

จากนั้นรีสตาร์ท service เพื่อให้โหลด config ใหม่:

```bash
sudo  systemctl restartduckyinc.service
```
![sudoedit](images/25.png)

### ⚙️ แก้ไข Unit File เพื่อ Privilege Escalation

- ทำการแก้ไขไฟล์ service เพื่อเพิ่มสิทธิ์ให้กับผู้ใช้ server-admin โดยให้สามารถรัน /bin/bash ด้วยสิทธิ์ root โดยไม่ต้องใช้รหัสผ่าน
- ใน ExecStart ของ service ให้เพิ่มคำสั่งนี้:

```bash
/bin/bash -c "echo 'server-admin ALL=(ALL) NOPASSWD: /bin/bash' | sudo tee /etc/sudoers.d/server_admin_as_root &&
```

![sudoedit](images/26.png)

- และอย่าลืมต่อท้าย app:app ด้วย

![sudoedit](images/27.png)

### 🔄 Reload และ Restart Service

- หลังจากแก้ไขไฟล์ .service ให้รันคำสั่งด้านล่างเพื่อให้ systemd โหลดค่าที่เปลี่ยนแปลง:

```bash
sudo systemctl daemon-reload
sudo systemctl restart duckyinc.service
```

![sudoedit](images/28.png)

> 📌 การที่เราทำแบบนี้คือจะทำให้ผู้ใช้ server-admin สามารถเรียกใช้งาน /bin/bash ด้วยสิทธิ์ root ได้โดยไม่ต้องกรอกรหัสผ่าน

- จากนั้นตรวจสอบสิทธิ์อีกครั้งด้วยคำสั่ง:

```bash
sudo -l
```
![sudo-l](images/29.png)
