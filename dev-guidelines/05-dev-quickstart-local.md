# 05 — Dev Quickstart: รัน Full Stack ใน Local

> เอกสารอ้างอิงสำหรับนักพัฒนา (developer reference) — วิธีรันระบบ vending-machine ทั้งกอง (DB + Redis + Mailpit → API → Web → Android) บนเครื่อง dev ตัวเดียว
> เอกสารนี้รองรับ **Gate G1 / Session 3** ตามแผนอบรม internal (Architecture + Stock + Debug) — น้องต้องรันทั้ง stack ขึ้นเองได้ก่อนเรียน flow
>
> ทุกคำสั่ง/ค่า env ในเอกสารนี้ถูก verify จาก source จริง (README, `package.json`, `docker-compose.yml`, `.env.example`, source ของ Kotlin app)
> จุดที่ยังยืนยันไม่ได้จะใส่ `⚠️ verify` กำกับไว้ — อย่าเดาคำสั่งเอง
>
> โครงสร้างระบบแบบเต็ม (REST/Socket/auth contract) อยู่ใน [`01-system-architecture.md`](01-system-architecture.md) — เอกสารนี้เน้นแค่ "วิธียกขึ้นมาให้รันได้"

---

## 1. ภาพรวม: ต้องรันอะไรพร้อมกันบ้าง

ต้องมี **4 ชั้น** ทำงานพร้อมกัน เรียงตาม dependency (ล่างขึ้นบน):

```
┌─────────────────────────────────────────────┐
│  Android kiosk (vending-machine-kotlin)      │  ← assembleDebug + ชี้ endpoint ไป API local
│  emulator: 10.0.2.2  /  device: LAN IP        │
└───────────────┬─────────────────────────────┘
                │ REST /api/v1/machine/* + Socket.IO /ws/machine (x-api-key)
┌───────────────▼─────────────────────────────┐
│  Web admin (vending-machine-web)             │  ← pnpm dev, VITE_API ชี้ไป API local
│  http://localhost:5173 (dev) — ดู §5         │
└───────────────┬─────────────────────────────┘
                │ REST /api/v1/portal/* (JWT)
┌───────────────▼─────────────────────────────┐
│  API (vending-machine-api)                   │  ← pnpm start:dev  (http://localhost:3000)
└───────────────┬─────────────────────────────┘
                │
┌───────────────▼─────────────────────────────┐
│  Infra (docker compose ใน vending-machine-api)│
│  MSSQL 1433 · Redis 6379 · Mailpit 8025/1025 │
└─────────────────────────────────────────────┘
```

### Port map (verify จาก source จริง)

| ส่วน | Port | ที่มา |
|---|---|---|
| API (NestJS) | `3000` | `.env.example` `PORT=3000`, README "Docker Services" |
| MSSQL Server 2022 | `1433` | `docker-compose.yml` service `mssql` |
| Redis | `6379` | `docker-compose.yml` service `redis` |
| Mailpit UI | `8025` | `docker-compose.yml` service `mail` |
| Mailpit SMTP | `1025` | `docker-compose.yml` service `mail` |
| Web dev server | `5173` (`vite` default) — `pnpm dev` ไม่ได้ fix port ⚠️ verify | `package.json` ของ web: `"dev": "vite --port 3000"` ← **ชนกับ API!** ดู §5 |

> ⚠️ **Port 3000 ชนกัน:** web `package.json` ตั้ง `"dev": "vite --port 3000"` ส่วน API ก็ใช้ 3000 เหมือนกัน หากรันทั้งคู่บนเครื่องเดียว vite จะหา port ว่างถัดไปให้เอง (เช่น 3001) หรือ error. แนะนำให้ override port ของ web ตอนรัน — ดู §5.

---

## 2. Clone repos + prerequisites

ก่อนเริ่ม ตรวจ prerequisites (Node 24, pnpm 10, Docker, JDK 17, Android SDK ฯลฯ) จาก
[`../pre-install-checklist.md`](../pre-install-checklist.md) ให้ครบก่อน

โครงทั้ง 3 repo อยู่ใต้ parent เดียวกัน (`vending-machine-ttfts/`):

```bash
# วางทุก repo ไว้ระดับเดียวกัน
vending-machine-ttfts/
├── vending-machine-api/      # backend (NestJS + Fastify)
├── vending-machine-web/      # admin SPA (Vite + React 18)
├── vending-machine-kotlin/   # Android kiosk
├── docs/                     # เอกสาร dev-only (PRD, แผนอบรม)
└── vending-machine-docs/     # เอกสารส่งมอบ (ไฟล์นี้อยู่ที่นี่)
```

Version ที่ต้องตรงกับ source (จาก `package.json`):

- **API:** Node `>=24`, pnpm `>=10` (`engines.pnpm = >=10`, pin `packageManager: pnpm@10.33.4`), Docker (สำหรับ MSSQL/Redis/Mailpit)
- **Web:** `packageManager: pnpm@10.33.4`, Node `>=24` (`.nvmrc = 24`, `engines.node = >=24`, pin `pnpm@10.33.4`)
- **Kotlin:** Java 17, Android SDK (`compileSdk=36`), AVD `vmkiosk` (1080×1920 @ 160dpi) — ดู `AGENTS.md` repo kotlin

---

## 3. ยก Infra ขึ้น (docker compose)

Infra (DB + Redis + Mailpit) มาพร้อมใน repo **vending-machine-api** ผ่าน `docker-compose.yml`

```bash
cd vending-machine-api
cp .env.example .env          # ต้องมี .env ก่อน — ดู §4
docker compose up -d mssql redis mail
```

> ใช้ `docker compose up -d mssql redis mail` เพื่อยกเฉพาะ **infra** (ไม่รวม service `api`) เพราะเราจะรัน API แบบ local watch mode ใน §4
> ถ้าอยากให้ Docker รัน API ให้ด้วย ใช้ `docker compose up` เฉย ๆ (compose จะ build + `migration:ci` + `start:prod` ให้อัตโนมัติ) — แต่จะไม่มี hot-reload

Service / image / port (จาก `docker-compose.yml` จริง):

| service | container_name | image | port |
|---|---|---|---|
| `mssql` | `vending-machine-mssql` | `mcr.microsoft.com/mssql/server:2022-latest` (`platform: linux/amd64`) | `1433:1433` |
| `redis` | `vending-machine-redis` | `redis:latest` (`--requirepass <redis-password>`) | `6379:6379` |
| `mail` | `vending-machine-mailpit` | `axllent/mailpit:latest` | `8025:8025`, `1025:1025` |

ค่าที่ compose ตั้งไว้ (ตรงกับ `.env.example` — ใช้ตอนเชื่อมจาก API local). placeholder ด้านล่างเป็นค่าตัวอย่าง — **ค่าจริงดูใน `vending-machine-api/docker-compose.yml` + `.env.example`**:

- MSSQL SA password: `<sa-password>`, DB ชื่อ `vending_machine_db` (มี healthcheck รอจน DB พร้อม)
- Redis password: `<redis-password>`, ใช้ DB index `1`
- Mailpit UI: เปิด `http://localhost:8025` ดูอีเมล (reset password ฯลฯ) — ไม่ต้อง config SMTP จริง

ตรวจว่าขึ้นครบ:

```bash
docker compose ps        # ทั้ง 3 ต้อง healthy
```

> **Apple Silicon (M-series):** image MSSQL fix `platform: linux/amd64` → รันผ่าน emulation, boot ช้า (`start_period: 40s`) เป็นปกติ รอ healthcheck ผ่านก่อน

---

## 4. รัน API local

ทำใน `vending-machine-api`:

```bash
cd vending-machine-api
pnpm install
cp .env.example .env          # ถ้ายังไม่ได้ทำใน §3
pnpm migration:up             # รัน migration (สร้าง schema + admin user)
pnpm seed:dev                 # seed dev data (optional แต่แนะนำ — ดูด้านล่าง)
pnpm start:dev                # watch mode → http://localhost:3000
```

> README แนะนำ `npm run ...` แต่ repo นี้ pin `pnpm@10.33.4` → **ใช้ `pnpm` ให้ตรง lockfile** (`pnpm-lock.yaml`) สคริปต์ใน `package.json` เหมือนกันทุกตัว

### env ที่ต้องตั้ง (key ที่สำคัญสุดสำหรับ local)

ค่า default ใน `.env.example` ชี้ไป **localhost** ตรงกับ infra ใน §3 อยู่แล้ว — แค่ `cp` ก็พอ ถ้าจะแก้ ระวัง 3 ตัวนี้:

| ตัวแปร | ค่า default (`.env.example`) | หมายเหตุ |
|---|---|---|
| `PORT` | `3000` | port ของ API |
| `DATABASE_URL` | `mssql://sa:<sa-password>@localhost:1433/vending_machine_db` | ตรงกับ compose |
| `REDIS_URL` | `redis://:<redis-password>@localhost:6379/1` | ตรงกับ compose |
| `SMTP_URL` | `smtp://mail:1025` | ⚠️ verify — ใน compose service ชื่อ `mail`; รัน API นอก Docker ต้องชี้ `smtp://localhost:1025` |
| `BASE_URL` | `http://localhost:8001/api/v1` | ⚠️ verify — ค่าใน `.env.example` คือ `:8001` แต่ใน compose env ของ `api` คือ `http://localhost:3000/api/v1`. ใช้ public URL ของ API (สำหรับ build ลิงก์ในอีเมล/ไฟล์) ให้ตั้งเป็น `http://localhost:3000/api/v1` ตอนรัน local |
| `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` | มีค่า dev ให้แล้ว | dev ใช้ได้เลย, prod ต้องเปลี่ยน |
| `MACHINE_CREATE_PASSWORD` | `<machine-create-password>` | รหัสตอนสร้างตู้ใหม่ (ใช้ตอน register machine + ออก api-key) |

> ⚠️ **`SMTP_URL` / `BASE_URL` host:** `.env.example` กับ `docker-compose.yml` ตั้งค่าต่างกัน (compose ใช้ service name `mail`/`mssql`/`redis`; `.env.example` ใช้ `localhost`). เมื่อรัน API **นอก** Docker (`pnpm start:dev`) ให้ใช้ `localhost` ทุก host. เมื่อรัน API **ใน** Docker ให้ใช้ service name. ตรวจค่า `BASE_URL`/`SMTP_URL` ใน `.env` ของน้องให้เป็น `localhost` ก่อนรัน local

### Seed / reset data

- `pnpm seed:dev` — seed dev data (1 machine, products, kanbans, mockup users) จาก `src/shared/database/seeds-dev/`
- script seed ทั้งหมด (จาก `package.json`): `seed:up` (prod seeds), `seed:dev`, `seed:e2e`, `seed:demo` — แต่ละตัวมีคู่ `:down` สำหรับ revert
- **ล้างข้อมูลการใช้งาน** (สินค้า/สต๊อก/ใบเบิก/ประวัติ) โดยคงค่าตั้งค่าระบบ: ใช้ SQL script
  (ใน repo vending-machine-api: `docs/reset-data.md`)
  (KEEP machines/users/themes/permissions, CLEAR products/stock/slips/history). **Backup ก่อนเสมอ** + หยุด API/Android/Web ก่อนรัน + FLUSH Redis หลังรัน (มีคำสั่งครบในเอกสารนั้น)

ตรวจว่า API ขึ้น:

```bash
curl http://localhost:3000/api/v1/health      # ต้องได้ {"ok":true}
```

> endpoint `/health` มาจาก `app.controller.ts` (`@Get('health')` → `{ ok: true }`)

---

## 5. รัน Web local

ทำใน `vending-machine-web`:

```bash
cd vending-machine-web
pnpm install
cp .env.example .env
```

แก้ `.env` ให้ `VITE_API` ชี้ไป API local (ค่า default ใน `.env.example` คือ `:4000` ซึ่ง**ไม่ตรง**กับ API port 3000):

```bash
# vending-machine-web/.env
VITE_API=http://localhost:3000/api/v1     # ← เปลี่ยนจาก :4000 ของ .env.example ให้ตรง API ใน §4
VITE_API_KEY=<api-key-ของตู้>             # ดูหมายเหตุด้านล่าง
VITE_DEV=true
```

| ตัวแปร | ที่มา / ค่า | หมายเหตุ |
|---|---|---|
| `VITE_API` | `.env.example` = `http://localhost:4000/api/v1` (fallback ใน `src/config/config.ts` ก็ `:4000`) | **ต้องแก้เป็น `:3000`** ให้ตรง API local |
| `VITE_API_KEY` | `.env.example` = `your_api_key_here` | api-key สำหรับ endpoint ที่ต้องใช้ (machine-scoped). สำหรับ login admin ปกติใช้ JWT ไม่ต้องมี key — ตั้งค่าเมื่อหน้าที่เรียกถึงต้องใช้ ⚠️ verify ว่าหน้าไหนบ้างที่บังคับ |

รัน dev server:

```bash
pnpm dev
```

> ⚠️ **Port:** `package.json` ตั้ง `"dev": "vite --port 3000"` — **ชนกับ API**. ให้ override port ตอนรันบนเครื่อง dev เดียวกัน:
> ```bash
> pnpm dev -- --port 5173        # เปิด http://localhost:5173
> ```
> (web README เขียนว่า "แอปจะรันที่ http://localhost:3000" ตาม script — แต่บน local ที่รัน API คู่กัน ต้องเลี่ยง port ชน) ⚠️ verify port สุดท้ายที่ทีมตกลงใช้

### override `.env` ชุด known-good สำหรับรัน Gate G1 (ทั้ง stack เครื่องเดียว)

ค่าชุดนี้ทดสอบแล้วว่ารันคู่กันได้บนเครื่อง dev ตัวเดียว (API 3000 / web 5173) — ใช้เป็นจุดตั้งต้นได้เลย แล้วค่อยปรับตามที่ทีมตกลง (ทีม**ยังไม่ได้ fix port อย่างเป็นทางการ** — ดู §1/§5):

```bash
# vending-machine-api/.env  (override จาก .env.example)
BASE_URL=http://localhost:3000/api/v1     # ให้ตรง API port; .env.example default = :8001

# vending-machine-web/.env  (override จาก .env.example)
VITE_API=http://localhost:3000/api/v1     # ให้ตรง API; .env.example default = :4000

# รัน web เลี่ยง port ชน API
pnpm dev -- --port 5173                    # → http://localhost:5173
```

เปิดเบราว์เซอร์ที่ port ของ web → หน้า login ของ admin portal

---

## 6. รัน Android (kiosk)

ทำใน `vending-machine-kotlin`. รายละเอียด build/AVD อยู่ใน `AGENTS.md` ของ repo — สรุปเฉพาะที่เกี่ยวกับ "ชี้ไป API local":

```bash
cd vending-machine-kotlin
./gradlew assembleDebug                  # build debug APK
# ติดตั้งลง emulator/device แล้วเปิดแอป
```

### ชี้ base URL ไป API local

**สำคัญ: app ไม่ได้ hardcode `BASE_URL`** — endpoint ถูก config ตอน runtime แล้วเก็บใน DataStore (`AppPreferences.API_ENDPOINT`). ตั้งผ่าน UI:

- ในแอป → **Settings** หรือ **ApiKeyDialog** (จากหน้า Home) → กรอก **API endpoint** + **api-key** ของตู้
- `DynamicBaseUrlInterceptor` จะ rewrite ทุก request ไปยัง endpoint ที่ save ไว้ (normalize ลงท้าย `/api/v1`)

**Emulator vs Device — host ที่ใช้ต่างกัน:**

| รันบน | endpoint ที่กรอก | เหตุผล |
|---|---|---|
| **Emulator (AVD)** | `http://10.0.2.2:3000` (หรือพิมพ์ `http://localhost:3000`) | `10.0.2.2` = alias ของ host loopback บน Android emulator. **debug build จะ rewrite `localhost`/`127.0.0.1` → `10.0.2.2` ให้อัตโนมัติ** (ดู `util/DevEndpointResolver.kt`) → พิมพ์ `localhost` ก็ใช้ได้ใน debug |
| **Device จริง (LAN)** | `http://<LAN-IP-ของเครื่อง-dev>:3000` เช่น `http://192.168.1.50:3000` | device คนละเครื่องกับ API → ต้องใช้ IP จริงของเครื่อง dev บน LAN เดียวกัน (ไม่ใช่ localhost/10.0.2.2) |

> รูปภาพ/media URL ที่ server ส่งมาเป็น `http://localhost:4000/...` จะถูก `util/UrlRewriter.kt` rewrite ให้ใช้ host เดียวกับ endpoint ที่ตั้งไว้ → emulator โหลดรูปได้ ⚠️ verify ว่า media host ตรงกับ API port (3000) ที่ใช้จริง

### Serial mock (`USE_MOCK`)

- เครื่อง dev ไม่มีฮาร์ดแวร์ serial → ใช้ mock dispense
- `SerialService.USE_MOCK` **default = `BuildConfig.DEBUG`** (debug build = mock อัตโนมัติ; toggle true/false ได้ตอน runtime)
- **release build บังคับ `false` เสมอ** — setter clamp เป็น false ถ้าไม่ใช่ `BuildConfig.DEBUG` (`_useMock = if (BuildConfig.DEBUG) value else false`) เพื่อกันไม่ให้ mock หลุดขึ้น fleet. เคยมีรอบที่ commit `7224090` flip เป็น plain `var` แล้ว ship mock ขึ้นทั้ง fleet จึงเพิ่ม guard clamp กัน silent regression (ดู comment ใน `SerialService.kt`)
- debug dispense flow: `adb logcat | grep VMC_SERIAL`

---

## 7. Smoke test (ทุกชั้นต่อกันแล้ว)

ทำตามลำดับ ทั้ง 4 ชั้นต้องผ่าน:

1. **Infra** — `docker compose ps` → `mssql` / `redis` / `mail` ทั้งหมด healthy
2. **API** — `curl http://localhost:3000/api/v1/health` → `{"ok":true}`
3. **Web login** — เปิด web (`http://localhost:5173` หรือ port ที่ตั้ง) → login admin
   - user: `admin` / email `admin@ttfts.com` (default seed user จาก migration `seed-user`)
   - password: ⚠️ verify — เป็น bcrypt hash ใน source หา plaintext ไม่ได้ → ขอจาก team lead หรือ reset ผ่านหน้า forgot-password แล้วรับลิงก์ใน Mailpit (`http://localhost:8025`)
4. **ตู้ online** — เปิดแอป Android → ตั้ง endpoint (`10.0.2.2:3000` หรือ LAN IP) + api-key ของตู้ → ตู้ต่อ Socket.IO สำเร็จ
   - ใน Web ไปหน้า **Monitor** → ตู้ขึ้น **online** (live socket map)
   - หน้า **Manage** ใช้ `lastHeartbeatAt` 90s — ทั้งคู่ online แปลว่า api-key ถูกต้อง (อ้างอิง `01-system-architecture.md`)
   - ลอง dispense 1 รายการ (mock) → ดู `adb logcat | grep VMC_SERIAL`

> ยังไม่มีตู้ในระบบ? สร้างตู้ใหม่ผ่าน Web admin (ใช้ `MACHINE_CREATE_PASSWORD=<machine-create-password>`) เพื่อให้ระบบออก api-key ให้ — แล้วเอา key นั้นไปกรอกในแอป Android (§6). seed `seed:dev` มี machine 1 ตัวแต่ใช้ api-key hash dummy (`$2b$10$dummyApiKeyHash...`) ที่ไม่มี plaintext → ต้องสร้างตู้ใหม่เพื่อได้ key ที่ใช้จริง ⚠️ verify

---

## เอกสารอ้างอิง

- แผนอบรม internal — Gate G1 / Session 3 อ้างอิงแผนนี้ (อยู่ในชุดเอกสาร dev)
- [`../pre-install-checklist.md`](../pre-install-checklist.md) — prerequisites ที่ต้องลงก่อน clone
- [`01-system-architecture.md`](01-system-architecture.md) — โครงสร้างระบบ + integration contract (REST/Socket/auth, port, x-api-key)
- [`02-debugging-runbook.md`](02-debugging-runbook.md) — debug server + ตู้ (adb logcat, FOOTER_LOGO)
- SQL ล้างข้อมูลการใช้งาน (คงค่าตั้งค่า) — ใน repo vending-machine-api: `docs/reset-data.md`
- `vending-machine-api/README.md` · `vending-machine-web/README.md` — README ต้นทางของแต่ละ repo
- `vending-machine-kotlin/AGENTS.md` — build/AVD/serial mock ของ Android kiosk
