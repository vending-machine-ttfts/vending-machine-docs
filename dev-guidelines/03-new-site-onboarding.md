# 03 — New-Site / Customer Onboarding Runbook (White-Label)

> เอกสารอ้างอิงสำหรับนักพัฒนา (developer reference) — วิธี onboard ลูกค้า/แบรนด์ใหม่แบบ white-label
> เอกสารนี้รองรับ Session "Deploy/OTA + เพิ่มลูกค้า/เครื่องใหม่" (ดูแผนอบรม internal)
>
> ทุกข้อมูลถูก verify จาก source จริง (script `.ps1` + README ใน `vending-machine-gitops`, web/api/kotlin repos)
> จุดที่ยังยืนยันไม่ได้จะใส่ `⚠️ verify` กำกับไว้
>
> **อย่า duplicate ขั้นตอน server:** การ setup ฝั่ง server แบบละเอียด (กรณี A ใหม่ / กรณี B เพิ่ม site) อยู่ใน
> [../tech-team-handbook.md](../tech-team-handbook.md) §1 แล้ว — เอกสารนี้เน้น **มุม onboarding ครบวงจร**: server-site +
> branding + เครื่อง + offline variant ของลูกค้า <customer>

---

## 1. ภาพรวม White-Label Model

หนึ่ง Windows Server รันได้ **หลายแบรนด์/ลูกค้า (site)** พร้อมกัน แยกกันด้วย environment variable ของ NSSM service
(อ้างอิง `README-White-Label-Windows-Services.md`)

### Shared (ติดตั้งครั้งเดียวต่อเครื่อง — ทุก site ใช้ร่วมกัน)

| ของที่แชร์ | รายละเอียด |
|---|---|
| **Redis** | Windows service (NSSM) `redis` bind `0.0.0.0:6379` — แยก site ด้วย `REDIS_DB` index (เช่น <brand-a>=1, <brand-b>=2) |
| **Mailpit** | Windows service (NSSM) `mailpit` — SMTP `1025`, Web UI `8025` |
| **Docker network** | `gitops_default` (driver `nat`) — ใช้ network เดียวกันทุก site (Windows สร้างหลาย NAT network ไม่ได้ HNS จะค้าง) |
| **SQL Server** | instance เดียว (`MSSQLSERVER:1433`) — แต่ **1 database ต่อ site** (ไม่ใช่ shared DB) |
| **IIS / ARR** | server-level proxy + URL Rewrite ตั้งครั้งเดียว — แต่ละ site เป็น IIS site แยก |

### Per-site (ทำซ้ำต่อแบรนด์)

| ของแยกต่อ site | ตัวอย่าง |
|---|---|
| **GitOps dir** | `C:\gitops-<brand>` (clone gitops repo เดียวกัน) |
| **NSSM deploy service** | `<brand>-gitops-deploy` (รัน `deploy.ps1`) |
| **Database** | `<brand>_db` |
| **Secret file** | `<gitops-dir>\.env.secrets` (+ `.env.docker`) — server-only, ห้าม commit |
| **IIS site** | `<BRAND>` → `C:\inetpub\wwwroot\<BRAND>\web.config` (host header แยก) |
| **Compose project / containers** | `<brand>-blue` / `<brand>-green`, `<brand>-vending-machine-web` / `-api` |
| **Ports (Blue/Green web+api)** | <brand-a> 8080-8083, <brand-b>/newco 9080-9083 — **ต้องไม่ชนกัน** |
| **Log dir** | `C:\logs\<brand>` |
| **Redis DB index** | unique ต่อ brand บน Redis ตัวเดิม |

> **กฎสำคัญ:** ทุก site clone **gitops repo เดียวกัน** ความต่างมาจาก `AppEnvironmentExtra` ของ NSSM service ล้วน ๆ
> (`COMPOSE_PROJECT_PREFIX`, port, `IIS_CONFIG_OUT`, ฯลฯ) — `deploy.ps1` render template ตามตัวแปรเหล่านี้

---

## 2. ขั้นตอน Add Site ใหม่ (Runbook)

> **Prerequisite:** server ผ่านกรณี A แล้ว (Docker, Git, NSSM, IIS+ARR via `setup-server.ps1`) และ Redis+Mailpit ขึ้นแล้ว
> (`setup-windows-services.ps1`). ถ้ายังไม่ครบ → ทำ [../tech-team-handbook.md](../tech-team-handbook.md) §1 (server setup) กรณี A ก่อน

**ทางลัด:** `setup-site.ps1` ทำ **กรณี B ทั้งชุด** ในคำสั่งเดียว — clone repo, สร้าง secret stub, upload/log dir,
Docker network, database (optional), IIS site + app pool, NSSM deploy service พร้อม environment ครบ
(idempotent — รันซ้ำ brand เดิมได้)

### 2.1 รัน setup-site.ps1 (PowerShell as Administrator)

พารามิเตอร์จริงจาก `param()` block ของ `setup-site.ps1`:

```powershell
pwsh -File C:\gitops-newco\setup-site.ps1 `
  -Brand newco `
  -Domain newco.<your-domain> `
  -BlueWebPort 9080 -BlueApiPort 9081 -GreenWebPort 9082 -GreenApiPort 9083 `
  -RedisDb 2 `
  -GitHubPat <github-pat> `
  -DockerUser <docker-user> -DockerPass <docker-token> `
  -CertThumbprint <thumbprint> `
  -CreateDatabase
```

**สิ่งที่ script ทำ (8 ขั้น ตามลำดับใน source):**

1. **เช็ก port ว่าง** — ถ้า port ถูกถือไว้โดย container ของ brand เดียวกัน (กรณี re-run) ถือว่า OK; ถ้าโดน process อื่น → throw
2. **GitOps dir** — `C:\gitops-<brand>` (default), clone จาก `-RepoUrl` (default = gitops repo); ถ้ามี `-GitHubPat` จะฝัง PAT ใน remote URL ให้ SYSTEM account pull ได้; สร้าง `upload_data\` + log dir
3. **Secret files** — เขียน `.env.docker` (ถ้าใส่ `-DockerUser`/`-DockerPass`); สร้าง `.env.secrets` จาก `.env.secrets.example` โดย **เติม `BASE_URL` / `DATABASE_NAME` / `REDIS_DB` / สุ่ม JWT ให้อัตโนมัติ** — ที่เหลือ (`DB_PASSWORD`, `REDIS_PASSWORD`, keys) ต้องกรอกเอง
4. **Docker network** — สร้าง `gitops_default` (driver `nat`) ถ้ายังไม่มี
5. **Database (optional)** — เฉพาะเมื่อใส่ `-CreateDatabase`: รัน `sqlcmd` `IF DB_ID(...) IS NULL CREATE DATABASE`; ถ้าไม่มี `sqlcmd` จะเตือนให้สร้างเอง
6. **IIS site + app pool** — สร้าง app pool (**No Managed Code**), site `<BRAND>` host header `-Domain` port 80; ถ้ามี `-CertThumbprint` เพิ่ม HTTPS binding 443 + **SNI**
7. **NSSM deploy service** — `<brand>-gitops-deploy` รัน `deploy.ps1` พร้อม `AppEnvironmentExtra` ครบ (`COMPOSE_PROJECT_PREFIX`, ports, `IIS_CONFIG_OUT`, ฯลฯ)
8. **Start** — เฉพาะเมื่อใส่ `-StartService` **และ** `.env.secrets` ไม่มี placeholder เหลือ (`your_*_here` / `generate_a_64`) — ไม่งั้นข้ามแล้วเตือน

> **default ที่ derive อัตโนมัติเมื่อไม่ระบุ:** `DatabaseName=<brand>_db`, `GitOpsDir=C:\gitops-<brand>`,
> `LogDir=C:\logs\<brand>`, `IisSiteName=<BRAND>` (uppercase), service `<brand>-gitops-deploy`,
> network `gitops_default`, NSSM `C:\nssm\win64\nssm.exe`, pwsh `C:\Program Files\PowerShell\7\pwsh.exe`

> **`-Brand` validation:** ต้องตรง pattern `^[a-z0-9][a-z0-9-]{1,20}$` (lowercase, ยาว 2-21 ตัว)

### 2.2 กรอก secrets ที่เหลือ — init-secrets.ps1 (แนะนำ)

`setup-site.ps1` สร้าง `.env.secrets` ให้บางส่วน แต่ `DB_PASSWORD`, `REDIS_PASSWORD`, `VITE_API_KEY`,
`MACHINE_CREATE_PASSWORD` ฯลฯ ยังว่าง ใช้ `init-secrets.ps1` กรอกแบบ interactive (มี help ทุก field, แสดงค่าที่พิมพ์
แบบ as-typed ไม่ mask เพื่อให้ตรวจก่อนกด Enter, auto-gen JWT):

```powershell
pwsh -File C:\gitops-newco\init-secrets.ps1 -Brand newco -Domain newco.<your-domain>
# overwrite ของเดิม (เก็บ .bak): เพิ่ม -Force
```

field สำคัญที่ต้องกรอก (จาก `init-secrets.ps1` field specs):

| Key | หมายเหตุ |
|---|---|
| `DB_HOST` / `REDIS_HOST` / `SMTP_HOST` | default = `$env:COMPUTERNAME` (host MACHINE NAME) — **`host.docker.internal` ไม่ resolve บน Windows container** |
| `DB_PASSWORD` | รหัส sa / app login |
| `REDIS_PASSWORD` | **ต้องตรงกับ** `-RedisPassword` ที่ใส่ตอน `setup-windows-services.ps1` (default `<redis-password>`) |
| `MACHINE_CREATE_PASSWORD` | รหัสที่ต้องกรอกตอนสร้างเครื่องใหม่ในเว็บ (ดู §4) |
| `VITE_API_KEY` | api-key ที่เว็บส่งให้ API |
| `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` | auto-gen 32-byte hex |
| `TURNSTILE_*` / `LOCIZE_*` / `ELASTIC_APM_*` | optional ตามที่ site ใช้ |
| `DOCKER_USER` / `DOCKER_PASS` | default user `<docker-user>`; pass = Docker Hub access token |

> **DB collation:** prod ใช้ `Thai_CI_AS` — ถ้าสร้าง DB เองนอก script ให้ `CREATE DATABASE <db> COLLATE Thai_CI_AS;`
> (กัน collation conflict ตอน query) ดู [../tech-team-handbook.md](../tech-team-handbook.md) §1 (server setup). **ไม่ต้องสร้างตาราง** — API container รัน TypeORM migration เองตอน start

### 2.3 Start + DNS

```powershell
C:\nssm\win64\nssm.exe start newco-gitops-deploy
# ชี้ DNS A/CNAME ของ newco.<your-domain> มา server นี้
```

ดู deploy loop ทำงาน: `Get-Content C:\logs\newco\newco-gitops-deploy.plain.log -Tail 30`
(ต้องเห็น git pull → compose pull → up → blue-green switch)

### 2.4 Observability (optional แต่แนะนำ)

รัน `setup-observability-services.ps1` **แยกครั้งละหนึ่ง brand** (NSSM service เป็น global) — ต้องครอบ
**ทั้ง blue และ green** ไม่งั้น log หายจาก Kibana เงียบ ๆ ตอน deploy สลับสี ดู [../tech-team-handbook.md](../tech-team-handbook.md) §1 (server setup / observability)

---

## 3. Branding Setup (โลโก้ + สีแบรนด์)

branding เป็น **per-machine** (ตาราง `machine_themes` 1 row ต่อเครื่อง) ไม่ใช่ per-brand global — ตั้งจากหน้า
web admin theme ของแต่ละเครื่อง แล้ว push เข้าตู้ผ่าน socket

### 3.1 Where stored

| ของ | ที่เก็บ |
|---|---|
| **โลโก้** | ตาราง `machine_themes` คอลัมน์ `company_image_id` (FK → ตาราง `files` ที่เก็บไฟล์จริง) |
| **สีแบรนด์** | ตาราง `machine_themes` คอลัมน์ `background_color` (`varchar(7)` hex `#RRGGBB`) |
| **อื่น ๆ** | `font_color`, `footer_type` (`custom`/`image`), `company_name`, `company_contact`, `footer_image_id` |

ตารางถูกสร้างใน migration (ใน repo vending-machine-api): `src/shared/database/migrations/1768422578854-init.ts`

### 3.2 How set (web admin)

- **หน้า:** Theme form ที่ `/machines/themes`
  (`vending-machine-web/src/components/features/Theme/ThemeForm.tsx`)
- **โลโก้:** upload field (folder `themes`) → ได้ `companyImageId`
- **สี:** RGB slider/picker (form-local `backgroundColorR/G/B`) → แปลงเป็น hex ตอน submit
- **API:** `PUT /api/v1/portal/themes/machines/{machineId}` ด้วย `UpdateMachineThemeDto`
  (field: `backgroundColor` hex, `companyImageId` UUID, `fontColor`, `footerType`, ฯลฯ)
  controller: `vending-machine-api/src/api/portal/theme.controller.ts`

### 3.3 How applied (kiosk)

1. API บันทึก `machine_themes` + emit socket event **`machine:themes:submitted`** (enum `MachineSocketEvent.THEMES_SUBMITTED`; ตู้ฟังที่ `EVENT_THEME_SUBMITTED`)
2. ตู้รับ event → `SyncRepository.syncThemeFromPayload(dto)` → เก็บลง `AppPreferences`
   (`setBackgroundColor`, `setCompanyImageUrl`, `setFontColor`, ฯลฯ)
3. **สี:** `generateBrandShades(hex)` สร้าง shade 50-900 จาก hex → `buildBrandColorScheme()`
   ใน `ui/theme/Theme.kt` map `primary = shade500`, `primaryContainer = shade100`,
   `onPrimaryContainer = shade900` — **default Orange `#E98932`** ถ้าไม่มีสีจาก server
4. **โลโก้:** `Footer` composable (`ui/components/Footer.kt`) แสดง `companyImageUrl` ผ่าน Coil `AsyncImage`
   เมื่อ `footerType != "image"` และ URL ไม่ว่าง

> **debug โลโก้หาย:** ตู้ไม่มี log ใน Kibana — ใช้ `adb logcat | grep FOOTER_LOGO` (เพิ่มมาเพื่อ diagnose โดยเฉพาะ)
> รายงาน "logo หาย" ส่วนใหญ่เป็น transient (data ปกติ) ดู debugging-runbook

---

## 4. เพิ่มเครื่อง/ตู้ (Machine)

### 4.1 Web admin — สร้างเครื่อง

- **หน้า:** `/machines/manage` → ปุ่ม create เปิด `MachineModal` (mode create)
  (`vending-machine-web/src/routes/(app)/machines/manage/index.tsx`)
- **กรอก:** `warehouse` (เลือกคลัง), `machineName`, `password`
- **`password`** = ค่า env `MACHINE_CREATE_PASSWORD` (ตั้งใน `.env.secrets` §2.2) — API validate ที่
  `POST /portal/machines` (`vending-machine-api/.../machine.controller.ts`): ถ้าไม่ตรง → `ForbiddenException('Invalid password')`
- **ผลลัพธ์:** `MachineCreatedResponseDto` คืน `id`, `code`, **`apiKey`** — เว็บโชว์ modal พร้อม api-key +
  endpoint URL (copy/print ได้) เว็บยัง gen **QR code** ที่เข้ารหัส `{apiKey}:{endpoint}` ไว้ด้วย **แต่แอป
  Android ปัจจุบันยังอ่าน QR นี้ไม่ได้** (ไม่มี camera/QR pairing scanner — ดู §4.2) → ตอนนี้ใช้ได้แค่ copy
  ค่าไปพิมพ์เองในตู้

### 4.2 Machine-side — ตั้งค่าแอป Android ที่ตู้

- **หน้า:** Settings tab (`ui/screens/settings/SettingsTabContent.kt`, ViewModel `SettingsViewModel.saveApiSettings(endpoint, key)`)
- **กรอกด้วยมือ (manual typing):** พิมพ์ "API Endpoint" + "API Key" จาก modal ของเว็บ (§4.1) ลงในช่อง
  → กด save → app sync data + reconnect socket
  > ตู้ **ไม่มี QR/camera scanner สำหรับ pairing** — `saveApiSettings()` รับเป็น string ล้วน ๆ
  > (HID barcode scanner ที่มีในแอปใช้สแกน *สินค้า* เท่านั้น ไม่ใช่ pairing) ดังนั้นต้องพิมพ์เอง
- **เก็บลง DataStore** (`data/preferences/AppPreferences.kt`):
  `api_endpoint`, `api_key`, `machine_id`, `machine_code`, `machine_name`
- **Base URL:** `DynamicBaseUrlInterceptor` อ่าน `api_endpoint` แล้ว normalize ให้ลงท้าย `/api/v1` เสมอ
- **Headers (`ApiKeyInterceptor`):** ทุก request แนบ
  - `x-api-key: <apiKey>` — machine api-key
  - `x-machine-user-id: <userId>` — user ที่ login (API `ApiKeyGuard` ใช้ระบุผู้ทำรายการ; ไม่มี → ได้ UUID สุ่ม → pickup unauthorized)
- **Socket:** `MachineSocketService` ต่อ `/ws/machine` แนบ header `x-api-key` เช่นกัน

> **Future enhancement:** ให้แอปสแกน QR `{apiKey}:{endpoint}` จากเว็บ (§4.1) เพื่อ fill endpoint+apiKey
> ทีเดียว — ยังไม่ทำ (ปัจจุบันต้องพิมพ์เอง ดูด้านบน) ถ้าจะเพิ่มต้องใส่ camera/QR reader แล้ว parse แยกที่ `:`

---

## 5. ลูกค้า Offline (first-setup-online) — "first-setup-online แล้วตัดเน็ต"

> สถานการณ์จริงของลูกค้า <customer> (ตามที่ตกลงกับลูกค้า): เครื่อง server เป็น Windows PC ใหม่ที่ IT ของลูกค้า (<IT contact>)
> ลง **SQL Server native** ไว้ให้แล้ว และ **มีอินเทอร์เน็ตตอน setup ครั้งแรก** (วัน demo หน้างาน)
> → `docker compose pull` ทำงานปกติเหมือน site online ทั่วไป **ไม่ต้องทำ docker save/load หรือ USB image bundle**

หลัง handover เครื่องจะถูก **ตัดเน็ต** → git pull / compose pull จะ fail ทุก tick **แต่ไม่เป็นไร** (harmless)

### ต่างจาก site online ทั่วไปตรงไหน

| | online ทั่วไป (<brand-a>/<brand-b>) | ลูกค้า <customer> (first-setup-online) |
|---|---|---|
| SQL Server | native บนเครื่อง (instance `MSSQLSERVER:1433`) | native — <customer> IT ลงให้ก่อนแล้ว (เหมือนกัน) |
| ตอน setup ครั้งแรก | มีเน็ต → `docker compose pull` ดึง web/api จาก Docker Hub | **มีเน็ต** → pull ได้ปกติ (ทำ §2 ครบในช่วงนี้) |
| หลัง setup เสร็จ | มีเน็ตต่อ → deploy ใหม่ได้เรื่อย ๆ | **ตัดเน็ต** → pull fail ทุก tick แต่ service ยังรันจาก image ที่ pull ไว้แล้ว |
| `.env.docker` | จำเป็น (login Docker Hub ตอน pull) | จำเป็น **เฉพาะช่วง setup ที่ยังมีเน็ต** |

### สิ่งที่ต้องทำให้ครบ "ในช่วงที่ยังมีเน็ต" (setup window)

ทำ §2 ตามปกติ — `setup-site.ps1` → กรอก secrets → start deploy service → ปล่อยให้ loop `docker compose pull`
ดึง `<docker-user>/vending-machine-web:<tag>` + `<docker-user>/vending-machine-api:<tag>` ลงมาให้ครบ + blue-green switch สำเร็จ
อย่างน้อย 1 รอบ **ก่อน** ตัดเน็ต ตรวจด้วย `doctor.ps1` (§6.1) ให้ผ่านหมดตอนยังออนไลน์

### หลังตัดเน็ต — พฤติกรรมที่คาดไว้ (ไม่ต้องแก้อะไร)

- `deploy.ps1` ยัง tick ทุก 60 วิ → `git pull` / `docker compose pull` **fail** แต่ script ทนได้
  (deploy ใหม่จะเกิดเฉพาะตอน rendered-compose MD5 เปลี่ยน ซึ่งมาจาก git pull เท่านั้น → ออฟไลน์ = ไม่ redeploy)
- container web/api **ยังรันต่อ** จาก image ที่ pull ไว้แล้ว → เว็บ + ตู้ใช้งานได้ตามปกติ
- สิ่งที่ทำไม่ได้หลังตัดเน็ต (เป็นเรื่องปกติของ customer offline): **deploy web/api เวอร์ชันใหม่** และ **Android OTA**
  (kiosk ดึง APK ใหม่ไม่ได้) — ดู [04-android-release-ota.md](04-android-release-ota.md)

> ขั้นตอนเต็ม + checklist ตัดเน็ต อยู่ใน runbook แยก: offline bring-up (ลูกค้า offline) — ดูในชุดเอกสาร internal

---

## 6. Verify Checklist

### 6.1 doctor.ps1 (read-only health check)

```powershell
pwsh -File C:\gitops-newco\doctor.ps1 -Brand newco `
  -BlueWebPort 9080 -BlueApiPort 9081 -GreenWebPort 9082 -GreenApiPort 9083
```

ครอบคลุม (จาก source): tooling (docker/git/pwsh/NSSM) · IIS+ARR+URL Rewrite · services (docker/sqlserver/redis/mailpit/W3SVC) ·
shared ports (1433/6379/1025/8025/80/443) · firewall (container→host) · network `gitops_default` driver `nat` ·
brand: gitops dir, `.env.secrets` (เช็ค placeholder เหลือไหม), IIS site + https binding, deploy service,
containers `<brand>-*`, **active slot + health endpoint** `http://localhost:<apiPort>/api/v1/health` (เช็ค `ok:true`)

> exit 0 = ไม่มี FAIL, exit 1 = มี FAIL อย่างน้อย 1; standby slot ไม่ respond = ปกติของ blue-green (ไม่ FAIL)

### 6.2 เปิดเว็บได้

```powershell
curl http://localhost:9080/                                  # web container (blue)
curl http://localhost:9081/api/v1/health                     # api health
curl -H "Host: newco.<your-domain>" http://127.0.0.1/        # ผ่าน IIS reverse proxy
```

แล้วเปิด `https://newco.<your-domain>` ในเบราว์เซอร์ → login / สร้าง admin user แรก (seed default คือ `admin@ttfts.com`)

### 6.3 เครื่อง online ใน Monitor

- ตั้งค่าตู้ (§4.2) → ตู้ต่อ socket สำเร็จ → ขึ้น **online ใน Monitor page** ของเว็บ
- ตู้ดึง branding (สี+โลโก้) + products + translations ลงมา
- Monitor page = live socket map; Manage page = `lastHeartbeatAt` 90s — ทั้งคู่ online แปลว่า api-key ถูกต้อง
- (ตู้จริง) ลองเบิก 1 รายการผ่าน serial: `adb logcat | grep VMC_SERIAL`

---

## เอกสารอ้างอิง

- แผนอบรม internal — Session "Deploy/OTA + เพิ่มลูกค้า/เครื่องใหม่" (ดูในชุดเอกสาร internal)
- [`../tech-team-handbook.md`](../tech-team-handbook.md) — §1 setup server กรณี A (ใหม่) / กรณี B (เพิ่ม site) แบบละเอียด + troubleshooting
- [`01-system-architecture.md`](01-system-architecture.md) — โครงสร้างระบบ + integration contract (REST/Socket/auth)
- [`02-debugging-runbook.md`](02-debugging-runbook.md) — debug server + ตู้ (adb logcat, blue-green, FOOTER_LOGO)
- `vending-machine-gitops/setup-site.ps1` / `init-secrets.ps1` / `doctor.ps1` — script onboarding
- `vending-machine-gitops/README-White-Label-Windows-Services.md` — naming convention + per-brand NSSM env
- `vending-machine-gitops/README-GitOps-NSSM-Windows-Server-2025.md` / `README-Windows-Docker-IIS.md` — GitOps loop + IIS/Docker
