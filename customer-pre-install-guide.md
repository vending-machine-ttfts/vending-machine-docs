# เอกสารเตรียมความพร้อมก่อนวันอบรม — รายการโปรแกรมและวิธีติดตั้ง (Windows)

เรียน ทีมงานผู้เข้าอบรม

เพื่อให้การอบรมวันที่ 20 เป็นไปอย่างราบรื่น รบกวนติดตั้งโปรแกรมตามรายการด้านล่างให้เรียบร้อยก่อนวันอบรม
หากติดปัญหาขั้นตอนใด แจ้งทีมงานได้เลย หรือส่ง Remote Access (AnyDesk / TeamViewer / RDP) มาให้ ทีมงานจะเข้าไปตรวจสอบให้ล่วงหน้า

> **คำแนะนำ:** หลายโปรแกรมมีขนาดใหญ่ (Android Studio ~3–4 GB, Docker images รวม ~5 GB) ควรดาวน์โหลดล่วงหน้าอย่างน้อย 1–2 วัน

---

## ส่วนที่ 1 — เครื่อง Server (Windows Server)

เครื่องนี้จะใช้รันระบบ Vending Machine (API, Web) ผ่าน Docker และมี SQL Server เป็นฐานข้อมูล

> **สำคัญ — server ต่างจากเครื่อง dev:** เครื่อง server ใช้ **Docker Engine + Windows Containers (Process Isolation)** — **ไม่ใช่ Docker Desktop และไม่ใช้ WSL2** (เครื่อง Programmer ในส่วนที่ 2 ถึงจะใช้ Docker Desktop)
>
> ทีมงานมีสคริปต์ `setup-server.ps1` ที่ติดตั้ง **Docker Engine + Containers feature + NSSM + IIS + Git** ให้อัตโนมัติในวันอบรม รายการด้านล่างคือสิ่งที่ลูกค้าเตรียม/ตรวจไว้ก่อนได้ (ลง SQL Server เองเพราะขึ้นกับ license)

### 1.1 รายการ

| # | โปรแกรม | ใช้ทำอะไร |
|---|---------|-----------|
| 1 | Windows Server 2025 + เปิด feature **Containers** | host ของระบบ |
| 2 | **Docker Engine** (Windows Containers — ไม่ใช่ Docker Desktop) | รัน API / Web |
| 3 | **SQL Server 2017 ขึ้นไป** (prod รัน 2017) | ฐานข้อมูล — ติดตั้งตรงบนเครื่อง (ไม่ได้รันใน Docker) |
| 4 | Git | ดึงไฟล์ตั้งค่าระบบ (gitops) |
| 5 | NSSM | ตั้งบริการ Windows ให้ระบบทำงานอัตโนมัติ |
| 6 | IIS + URL Rewrite + ARR | ตัวกระจายเว็บ (reverse proxy) |

### 1.2 วิธีติดตั้ง

> ข้อ 1, 2, 4, 5, 6 — ถ้าให้ทีมงานรัน `setup-server.ps1` วันอบรม จะติดตั้งให้ครบอัตโนมัติ ลูกค้าเตรียมแค่ Windows Server + สิทธิ์ Administrator + อินเทอร์เน็ต (หรือแจ้งทีมงานถ้า offline)

**1) เปิด Windows Containers feature** — เปิด PowerShell แบบ *Run as Administrator* แล้วรัน:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Containers -All
```

รีสตาร์ตเครื่อง 1 ครั้งหลังติดตั้งเสร็จ

**2) Docker Engine (Windows Containers)** — ติดตั้งแบบ static binary + register เป็น Windows service (**ไม่ใช่ Docker Desktop, ไม่ใช้ WSL2**)

- ทีมงานติดตั้งให้ผ่าน `setup-server.ps1` (Docker Engine 28.x)
- ทดสอบหลังติดตั้ง: `docker version` → ฝั่ง **Server ต้องเป็น `windows/amd64`** และ `docker info` แสดง `Isolation: process`

**3) SQL Server (2017 ขึ้นไป)** — ติดตั้งตรงบนเครื่อง (ใช้ license ของลูกค้า) ไม่ได้รันใน Docker · prod ปัจจุบันรัน **SQL Server 2017** (default instance, port 1433)

**4) Git**

ดาวน์โหลด: <https://git-scm.com/download/win> → ติดตั้งแบบ Next ทั้งหมด (ค่าเริ่มต้น)
ทดสอบ: `git --version`

**5) NSSM**

1. ดาวน์โหลด: <https://nssm.cc/download> (เลือกรุ่นล่าสุด เช่น 2.24)
2. แตก zip แล้วคัดลอก `win64\nssm.exe` ไปไว้ที่ `C:\tools\nssm\nssm.exe` (สร้างโฟลเดอร์เอง)
3. ทดสอบ: `C:\tools\nssm\nssm.exe version`

**6) IIS + URL Rewrite + ARR** — เปิด PowerShell (Administrator):

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

จากนั้นติดตั้งโมดูลเพิ่ม 2 ตัว (ดาวน์โหลดแล้วกด Next ติดตั้งตามปกติ):
- URL Rewrite: <https://www.iis.net/downloads/microsoft/url-rewrite>
- Application Request Routing (ARR): <https://www.iis.net/downloads/microsoft/application-request-routing>

> ถ้าเครื่อง Server **ไม่มีอินเทอร์เน็ต** แจ้งทีมงานล่วงหน้า — ทีมงานจะเตรียม Docker images (Windows containers) และตัวติดตั้งทั้งหมดใส่ USB ไปในวันอบรมแทน

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

> **Node version manager (optional):** repo `api`/`web` pin Node ไว้ที่ `.nvmrc` + `.node-version` (= 24) แล้ว → ลง **fnm** หรือ **nvm** จะสลับเป็น Node 24 ให้อัตโนมัติเมื่อ `cd` เข้า repo (pnpm ถูก pin ผ่าน `packageManager` ใน package.json — corepack จัดการเวอร์ชันให้เอง)
> - **fnm** (<https://github.com/Schniz/fnm>) — เบา เร็ว **แนะนำ**
> - **nvm** (Windows ใช้ `nvm-windows`) — ใช้ได้ · ⚠️ หลัง `nvm use` ทุกครั้งต้อง `corepack enable` ใหม่ (shim ผูกกับ Node แต่ละตัว)
> - **Volta** ยังไม่ได้ตั้งใน repo (ต้องเพิ่ม `volta` field ก่อน) · asdf ไม่รองรับ Windows native
> ถ้าทำแค่โปรเจคนี้เครื่องเดียว → Node 24 LTS installer ตรงๆ ก็พอ

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

## ส่วนที่ 3 — ตู้ Vending (เครื่อง Android)

ตู้แบ่งเป็น 2 แบบ:

### 3.1 ตู้สำหรับอบรม / ทดสอบ — *Optional (ไว้ทีหลังได้)*

> ไม่ต้องเตรียมล่วงหน้าก็ได้ — ทีมงานจะตั้งค่าให้ในวันอบรม

1. เสียบสาย USB เชื่อมกับคอมพิวเตอร์ได้ (สายที่ส่งข้อมูลได้ ไม่ใช่สายชาร์จอย่างเดียว)
2. เปิด **Developer options**: Settings → About tablet → กด **Build number** 7 ครั้ง
3. เปิด **USB debugging**: Settings → System → Developer options → USB debugging = ON
4. ตัวแอปหน้าตู้ ทีมงานจะนำไปติดตั้งให้ในวันอบรม

### 3.2 ตู้ production (เครื่องหน้างานจริง) — **ต้องลง Tailscale**

ตู้ที่ติดตั้งใช้งานจริงต้องมี **Tailscale** เพื่อให้ทีมงาน remote เข้าดูแล/อัปเดตจากระยะไกลได้ (เพราะ server หน้างานเป็นแบบ offline)

ลูกค้าเตรียม:
- **บัญชี Tailscale** (สมัครฟรีด้วย Google/Microsoft/GitHub) — ใช้บัญชีเดียวกันทั้งฝั่งตู้และทีมงาน
- อนุญาตให้ตู้ออกอินเทอร์เน็ตได้ (Tailscale เชื่อมแบบ mesh ไม่ต้อง config router/port-forward)

> การติดตั้ง Tailscale + ตั้งค่า kiosk / Server URL / API key / SSL cert ทีมงานทำให้ในวันติดตั้ง (ดู `tech-team-handbook.md` ส่วนที่ 2)

---

ขอบคุณครับ
ทีมงาน TTFTS
