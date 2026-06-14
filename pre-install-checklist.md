# Checklist โปรแกรมที่ต้องติดตั้งล่วงหน้า (ก่อนวันอบรม 20)

อ้างอิง stack จริง: API = NestJS, Web = React/Vite (pnpm) — build เป็น **Windows containers (Process Isolation)** รันบน Docker Engine ที่ Windows Server; DB = **SQL Server 2022 (ติดตั้ง native ไม่ใช่ container)**; IIS reverse proxy, deploy ด้วย GitOps + NSSM

---

## 1) เครื่อง Server (Windows Server — ตัวที่จะไปอยู่หน้างาน/offline)

| # | โปรแกรม | หมายเหตุ |
|---|---------|----------|
| 1 | **Docker Engine** (Windows Containers, **Process Isolation**) | รัน API/Web — **ไม่ใช่ Docker Desktop / ไม่ใช้ WSL2 / ไม่ใช่ Linux containers** |
| 2 | **SQL Server 2022** | ฐานข้อมูล — ติดตั้ง native บนเครื่อง (ไม่ได้รันใน Docker) |
| 3 | **Git** | ใช้ดึง repo gitops / config |
| 4 | **NSSM** | ทำ Windows service สำหรับ deploy + log tail |
| 5 | **IIS** + โมดูล **URL Rewrite** + **ARR** | reverse proxy เข้า container (ตาม `web-iis-server.config`) |
| 6 | (มีอยู่แล้ว) PowerShell 5+ | รัน `deploy.ps1` / `setup-windows-services.ps1` |

> ข้อ 1, 3, 4, 5 ติดตั้งอัตโนมัติได้ด้วย `setup-server.ps1` (Docker Engine 28.x + Containers feature + NSSM + IIS + Git)

**Docker images ที่ต้องใช้** — **Windows containers** เท่านั้น (ถ้า server ไม่มี internet ให้เตรียม `docker save` ใส่ USB ไป `docker load` หน้างาน):
- `sangzn34/vending-machine-api:<tag>`
- `sangzn34/vending-machine-web:<tag>`

> prod compose รันแค่ **web + api** (Windows containers) เท่านั้น (ดู `docker-compose.template.yml`) · **SQL Server / Redis / Mailpit ไม่ได้รันใน Docker** — SQL Server = native install, Redis + Mailpit = **Windows service ผ่าน NSSM** (`setup-windows-services.ps1`, แชร์ทุก site) → ไม่ต้องเตรียม Linux image พวกนี้

**ขอจากฝั่งลูกค้า:** remote access (AnyDesk/TeamViewer/RDP) เข้า server เพื่อเช็กก่อนวันที่ 20

---

## 2) เครื่องคอม Programmer (ฝั่งลูกค้า — ไว้แก้โค้ด/debug)

| # | โปรแกรม | เวอร์ชัน/หมายเหตุ |
|---|---------|------------------|
| 1 | **Git** | + สิทธิ์เข้า repo |
| 2 | **Node.js** | v24 (ตรงกับ image ที่ build) |
| 3 | **pnpm** | v10.x (เปิดผ่าน `corepack enable` ได้) |
| 4 | **VS Code** | แก้โค้ด API/Web |
| 5 | **Docker Desktop** | รัน stack ทั้งชุดบนเครื่องตัวเอง (`docker compose up`) |
| 6 | **Android Studio** (Hedgehog ขึ้นไป) | สำหรับแอปตู้ (Kotlin) — ลงพร้อม Android SDK (API 35/36) + JDK 17 (มากับ Studio) |
| 7 | — สร้าง AVD `vmkiosk` | 1080×1920 @ 160dpi (ไว้เทสต์ UI ตู้) |
| 8 | **SSMS** หรือ **Azure Data Studio** หรือ **DBeaver** | ต่อ SQL Server ดูข้อมูล |
| 9 | **Postman** (หรือ Insomnia) | ยิงเทสต์ API |
| 10 | **scrcpy** *(optional)* | ดู/ควบคุมหน้าจอตู้ผ่าน adb |

> ข้อ 2–3 ข้ามได้ถ้าจะ build ผ่าน Docker อย่างเดียว แต่แนะนำลงไว้สำหรับรัน dev mode (`pnpm dev`)

---

## 3) ตู้ Android

แบ่งเป็น 2 แบบ — เตรียมไม่เหมือนกัน:

### 3a) ตู้สำหรับอบรม / dev — *Optional (ไว้ทีหลังได้)*

- เปิด **Developer options + USB debugging**
- สาย USB + ไดรเวอร์ ADB บนคอมที่จะ deploy
- APK ล่าสุด (จะ build ให้วันอบรม หรือเตรียม `app-release.apk` ไป)
- **ไม่ต้องลง Tailscale**

### 3b) ตู้ production (เครื่องหน้างานจริง) — **ต้องลง Tailscale** ⭐

- ทุกอย่างใน 3a + ด้านล่าง
- **Tailscale** — สมัคร account (Google/MS/GitHub) ใช้ login เดียวกันทั้งฝั่งตู้+ทีม, เตรียม APK จาก <https://pkgs.tailscale.com/stable/#android>, เปิด **Run on startup / Auto-connect**
- Device Owner (full kiosk) + Server URL + API key + SSL cert

> เหตุผล: server หน้างาน offline → Tailscale คือช่องเดียวที่ทีม remote เข้าตู้ได้ (ADB install / logcat / shell / reboot, push APK). Step เต็ม: `tech-team-handbook.md` §2

---

*โหลดทุกตัวล่วงหน้า — วันอบรมจะได้ไม่เสียเวลารอดาวน์โหลด โดยเฉพาะ Android Studio + SDK (~หลาย GB) และ Docker images*
