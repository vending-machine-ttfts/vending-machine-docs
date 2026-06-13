# Checklist โปรแกรมที่ต้องติดตั้งล่วงหน้า (ก่อนวันอบรม 20)

อ้างอิง stack จริง: API = NestJS (node:24-alpine), Web = React/Vite (pnpm), DB = SQL Server 2022, Redis, Mailpit — รันผ่าน Docker บน Windows Server, IIS reverse proxy, deploy ด้วย GitOps + NSSM

---

## 1) เครื่อง Server (Windows Server — ตัวที่จะไปอยู่หน้างาน/offline)

| # | โปรแกรม | หมายเหตุ |
|---|---------|----------|
| 1 | **Docker Desktop** (หรือ Docker Engine) | ตั้ง Linux containers (WSL2) — ใช้รันทุก service |
| 2 | **Git** | ใช้ดึง repo gitops / config |
| 3 | **NSSM** | ทำ Windows service สำหรับ deploy + log tail |
| 4 | **IIS** + โมดูล **URL Rewrite** + **ARR** | reverse proxy เข้า container (ตาม `web-iis-server.config`) |
| 5 | (มีอยู่แล้ว) PowerShell 5+ | รัน `deploy.ps1` / `setup-windows-services.ps1` |

**Docker images ที่ต้องใช้** (ถ้า server ไม่มี internet ให้เตรียม `docker save` ใส่ USB ไป `docker load` หน้างาน):
- `mcr.microsoft.com/mssql/server:2022-latest`
- `redis:latest`
- `axllent/mailpit:latest`
- `sangzn34/vending-machine-api:<tag>`
- `sangzn34/vending-machine-web:<tag>`

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

## 3) ตู้ Android (เครื่องใหม่ที่จะใช้สอน)

- เปิด **Developer options + USB debugging**
- สาย USB + ไดรเวอร์ ADB บนคอมที่จะ deploy
- APK ล่าสุด (จะ build ให้วันอบรม หรือเตรียม `app-release.apk` ไป)

---

*โหลดทุกตัวล่วงหน้า — วันอบรมจะได้ไม่เสียเวลารอดาวน์โหลด โดยเฉพาะ Android Studio + SDK (~หลาย GB) และ Docker images*
