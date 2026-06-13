# เอกสารเตรียมความพร้อมก่อนวันอบรม — รายการโปรแกรมและวิธีติดตั้ง (Windows)

เรียน ทีมงานผู้เข้าอบรม

เพื่อให้การอบรมวันที่ 20 เป็นไปอย่างราบรื่น รบกวนติดตั้งโปรแกรมตามรายการด้านล่างให้เรียบร้อยก่อนวันอบรม
หากติดปัญหาขั้นตอนใด แจ้งทีมงานได้เลย หรือส่ง Remote Access (AnyDesk / TeamViewer / RDP) มาให้ ทีมงานจะเข้าไปตรวจสอบให้ล่วงหน้า

> **คำแนะนำ:** หลายโปรแกรมมีขนาดใหญ่ (Android Studio ~3–4 GB, Docker images รวม ~5 GB) ควรดาวน์โหลดล่วงหน้าอย่างน้อย 1–2 วัน

---

## ส่วนที่ 1 — เครื่อง Server (Windows Server)

เครื่องนี้จะใช้ติดตั้งระบบ Vending Machine ทั้งชุด (Database, API, Web)

### 1.1 รายการ

| # | โปรแกรม | ใช้ทำอะไร |
|---|---------|-----------|
| 1 | WSL2 | ตัวกลางสำหรับรัน Docker |
| 2 | Docker Desktop | รันระบบทั้งหมด (Database / API / Web) |
| 3 | Git | ดึงไฟล์ตั้งค่าระบบ |
| 4 | NSSM | ตั้งบริการ Windows ให้ระบบทำงานอัตโนมัติ |
| 5 | IIS + URL Rewrite + ARR | ตัวกระจายเว็บ (reverse proxy) |

### 1.2 วิธีติดตั้ง

**1) WSL2** — เปิด PowerShell แบบ *Run as Administrator* แล้วรัน:

```powershell
wsl --install --no-distribution
```

รีสตาร์ตเครื่อง 1 ครั้งหลังติดตั้งเสร็จ

**2) Docker Desktop**

1. ดาวน์โหลด: <https://www.docker.com/products/docker-desktop/>
2. ติดตั้งโดยเลือก **Use WSL 2 instead of Hyper-V** (ค่าเริ่มต้น)
3. เปิด Docker Desktop ครั้งแรก รอจนสถานะมุมล่างซ้ายเป็นสีเขียว (Engine running)
4. ทดสอบ: เปิด PowerShell รัน `docker version` ต้องแสดงทั้ง Client และ Server

**3) Git**

ดาวน์โหลด: <https://git-scm.com/download/win> → ติดตั้งแบบ Next ทั้งหมด (ค่าเริ่มต้น)
ทดสอบ: `git --version`

**4) NSSM**

1. ดาวน์โหลด: <https://nssm.cc/download> (เลือกรุ่นล่าสุด เช่น 2.24)
2. แตก zip แล้วคัดลอก `win64\nssm.exe` ไปไว้ที่ `C:\tools\nssm\nssm.exe` (สร้างโฟลเดอร์เอง)
3. ทดสอบ: `C:\tools\nssm\nssm.exe version`

**5) IIS + URL Rewrite + ARR** — เปิด PowerShell (Administrator):

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

จากนั้นติดตั้งโมดูลเพิ่ม 2 ตัว (ดาวน์โหลดแล้วกด Next ติดตั้งตามปกติ):
- URL Rewrite: <https://www.iis.net/downloads/microsoft/url-rewrite>
- Application Request Routing (ARR): <https://www.iis.net/downloads/microsoft/application-request-routing>

> ถ้าเครื่อง Server **ไม่มีอินเทอร์เน็ต** แจ้งทีมงานล่วงหน้า — ทีมงานจะเตรียม Docker images และตัวติดตั้งทั้งหมดใส่ USB ไปในวันอบรมแทน

---

## ส่วนที่ 2 — เครื่องคอมพิวเตอร์ของ Programmer

เครื่องนี้ใช้สำหรับแก้ไขโปรแกรม ทดสอบ และ deploy (Windows 10/11)

### 2.1 รายการ

| # | โปรแกรม | เวอร์ชัน | ใช้ทำอะไร |
|---|---------|---------|-----------|
| 1 | Git | ล่าสุด | จัดการ source code |
| 2 | Node.js | **24 (LTS)** | รัน/พัฒนา API และ Web |
| 3 | pnpm | 10.x | ตัวจัดการ package |
| 4 | Visual Studio Code | ล่าสุด | แก้ไขโค้ด |
| 5 | Docker Desktop | ล่าสุด | รันระบบจำลองบนเครื่องตัวเอง |
| 6 | Android Studio | ล่าสุด | พัฒนาแอปหน้าตู้ (Kotlin) |
| 7 | SQL Server Management Studio (SSMS) | ล่าสุด | ดู/แก้ข้อมูลใน Database |
| 8 | Postman | ล่าสุด | ทดสอบ API |

### 2.2 วิธีติดตั้ง

วิธีที่เร็วที่สุด: เปิด PowerShell (ไม่ต้อง Administrator ก็ได้) แล้วรันทีละบรรทัด

```powershell
winget install Git.Git
winget install OpenJS.NodeJS.LTS
winget install Microsoft.VisualStudioCode
winget install Docker.DockerDesktop
winget install Google.AndroidStudio
winget install Microsoft.SQLServerManagementStudio
winget install Postman.Postman
```

> ถ้าเครื่องไม่มีคำสั่ง `winget` ให้ดาวน์โหลดติดตั้งเองจากเว็บทางการ:
> - Git: <https://git-scm.com/download/win>
> - Node.js 24: <https://nodejs.org/> (เลือก v24 LTS, Windows Installer .msi)
> - VS Code: <https://code.visualstudio.com/>
> - Docker Desktop: <https://www.docker.com/products/docker-desktop/> (ต้องลง WSL2 ก่อน — ดูส่วนที่ 1 ข้อ 1)
> - Android Studio: <https://developer.android.com/studio>
> - SSMS: <https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms>
> - Postman: <https://www.postman.com/downloads/>

**ติดตั้ง pnpm** (หลังลง Node.js เสร็จ) — เปิด PowerShell ใหม่ แล้วรัน:

```powershell
corepack enable
pnpm --version
```

ถ้าขึ้นเลขเวอร์ชัน 10.x ถือว่าเรียบร้อย

**ตั้งค่า Android Studio** (ทำหลังติดตั้งเสร็จ เปิดโปรแกรมครั้งแรก):

1. เปิด Android Studio → ทำตาม Setup Wizard จนจบ (จะดาวน์โหลด Android SDK อัตโนมัติ)
2. ไปที่ **Settings → Languages & Frameworks → Android SDK** → ติ๊กติดตั้ง **Android 15 (API 35)** และ **Android 16 (API 36)**
3. สร้าง Emulator สำหรับทดสอบหน้าจอตู้: **Device Manager → Add a new device → New hardware profile**
   - ชื่อ: `vmkiosk`
   - Screen: **1080 × 1920**, ความหนาแน่น **160 dpi (mdpi)**, Portrait
   - System image: Android 15 (API 35)

**ทดสอบความพร้อมรวม** — รันใน PowerShell แล้วถ่ายหน้าจอผลลัพธ์ส่งให้ทีมงาน:

```powershell
git --version
node --version
pnpm --version
docker version
adb version
```

> `adb` จะใช้ได้หลังติดตั้ง Android Studio — ถ้าไม่พบคำสั่ง ให้เพิ่ม `%LOCALAPPDATA%\Android\Sdk\platform-tools` เข้า PATH

---

## ส่วนที่ 3 — ตู้ Vending (เครื่อง Android ใหม่)

1. เสียบสาย USB เชื่อมกับคอมพิวเตอร์ได้ (เตรียมสาย USB ที่ใช้ส่งข้อมูลได้ ไม่ใช่สายชาร์จอย่างเดียว)
2. เปิด **Developer options**: Settings → About tablet → กด **Build number** 7 ครั้ง
3. เปิด **USB debugging**: Settings → System → Developer options → USB debugging = ON
4. ตัวแอปหน้าตู้ ทีมงานจะนำไปติดตั้งให้ในวันอบรม

---

ขอบคุณครับ
ทีมงาน TTFTS
