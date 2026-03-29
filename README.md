# 📄 JobBKK Resume Scraper (Playwright + Google Sheet)

ระบบดึงข้อมูลผู้สมัครจาก JobBKK (iCMS) อัตโนมัติ
พร้อมกรองเฉพาะ:

* Resume Score ≥ 80%
* อยู่ใน “ปทุมธานี / ลำลูกกา”
* อัพเดตไม่เกิน 6 เดือน

และส่งข้อมูลเข้า Google Sheet

---

# 🚀 Features

* Auto login (รองรับการดีด session เก่าออก)
* ดึงข้อมูลจากหน้า list + resume
* กรองข้อมูลอัตโนมัติ
* ส่งเข้า Google Sheet อัตโนมัติ
* ตั้งเวลา run อัตโนมัติได้ (Windows Task Scheduler)

---

# 🧰 Requirements

* Node.js (เวอร์ชัน 18 ขึ้นไป)
* Windows (สำหรับ Task Scheduler)

---

# ⚙️ 1. ติดตั้ง Node.js

ดาวน์โหลดและติดตั้ง:
👉 [https://nodejs.org/](https://nodejs.org/)

ตรวจสอบ:

```bash
node -v
npm -v
```

---

# ⚙️ 2. ดาวน์โหลดโปรเจค

แตกไฟล์ ZIP ที่ได้รับ แล้วเปิดโฟลเดอร์

---

# ⚙️ 3. ติดตั้ง dependencies

เปิด Command Prompt / Terminal ในโฟลเดอร์โปรเจค แล้วรัน:

```bash
npm install
npx playwright install
```

---

# ⚙️ 4. ตั้งค่าบัญชีและ Sheet

ข้อมูลต่อไปนี้จะถูกเตรียมไว้ให้แล้ว:

* Username / Password สำหรับ JobBKK
* Google Sheet (พร้อมใช้งาน)

👉 ให้เปิดไฟล์ `index.js` แล้วตรวจสอบค่าที่กำหนดไว้ให้เรียบร้อย

---

# ▶️ 5. ทดสอบรัน

```bash
node index.js
```

ถ้าทำงานถูก:

* browser จะเปิดอัตโนมัติ
* login เข้า JobBKK
* ดึงข้อมูลผู้สมัคร
* ส่งเข้า Google Sheet

---

# ⏰ 6. ตั้งเวลาอัตโนมัติ (Windows Task Scheduler)

## 6.1 เปิด Task Scheduler

กด Start → พิมพ์:

```
Task Scheduler
```

---

## 6.2 Create Task

### 🔹 General

* Name: `JobBKK Bot`

---

### 🔹 Trigger

* New Trigger
* Begin: On a schedule
* Daily
* เวลา: 07:00

---

### 🔹 Action

* Action: Start a program

Program/script:

```
node
```

Add arguments:

```
index.js
```

Start in:

```
C:\path\to\project
```

---

### 🔹 Settings

* Allow task to be run on demand ✔

---

## 6.3 ทดสอบ

คลิกขวา Task → Run

---

# ⚠️ Important Notes

* ต้องเปิดเครื่องไว้
* ถ้ามี session ซ้อนอยู่แล้ว ระบบจะ auto เตะออกให้

---

# 🧠 How It Works

```text
Playwright (Local Machine)
        ↓
Login + Scrape
        ↓
Send Data
        ↓
Google Sheet
```

---

# 🛠 Troubleshooting

## ❌ Login ไม่ได้

* ตรวจ username / password
* เช็คอินเทอร์เน็ต

---

## ❌ ไม่มีข้อมูลเข้า Sheet

* ตรวจว่าระบบรันจบหรือไม่ เช่น มีผู้อื่น login เข้ามาระหว่างรันระบบ
* เช็คว่า Script ไม่ error

---

## ❌ Task Scheduler ไม่รัน

* เช็ค path ให้ถูก
* เช็คว่า node ใช้ได้ใน cmd

---

# ✅ Done

ระบบพร้อมใช้งานจริง
สามารถนำไปใช้กับเครื่องอื่นได้ทันที 🚀
