# 02 — Debugging Runbook

> เอกสารนี้รองรับ **Session 3 (Online) — Architecture + Stock + Debug** ในแผนอบรม
> (ดูแผนอบรม internal).
> เป้าหมาย: น้องไล่หา root cause จาก log ของระบบ vending machine ได้ — รู้ว่า log แต่ละชนิด
> "อยู่ที่ไหน" และ "เข้าถึงยังไง".

---

## 1. แผนที่ Log — ที่ไหนเก็บอะไร

กฎเหล็กข้อเดียวที่ต้องจำให้ขึ้นใจ:

> **Kibana / ELK เก็บเฉพาะ server log (API) เท่านั้น.**
> **Log ของตู้ (kiosk) — dispense / serial / footer logo — อยู่ที่ `adb logcat` เท่านั้น
> ไม่ได้ขึ้น Kibana.**

ถ้าจำผิดข้อนี้ จะเสียเวลานั่งหา serial log ใน Kibana ทั้งวันโดยไม่เจออะไรเลย.

| แหล่ง log | เก็บอะไร | ไม่เก็บอะไร | เข้าถึงด้วย |
|-----------|----------|-------------|------------|
| **Kibana / ELK** | server-side API log (NestJS + Pino), index `logs-vending-machine-*` | log ของแอป Android | Kibana UI (internal) |
| **`adb logcat`** | log ของตู้ทั้งหมด — dispense, serial, footer logo, socket, sync | — | สาย USB หรือ Tailscale → ADB |
| **ไฟล์ log บนตู้ (durable)** | Timber ทุกบรรทัด เขียนลงไฟล์รายวัน `log-YYYY-MM-DD.txt` (เก็บ 14 วัน) — รอดทั้ง debug/release | — | `adb pull /sdcard/Android/data/com.vendingmachine.app/files/logs/` |
| **Mailpit** | outbound mail ที่ API ส่ง (dev/test) | mail production จริง | Mailpit UI พอร์ต `8025` |
| **Container log (Docker)** | stdout/stderr ของ container `vending-machine-api` / `web` | — | `docker compose logs` บนเซิร์ฟเวอร์ |
| **NSSM log-tail → Kibana** | ท่อส่ง container log ขึ้น ELK (per-brand, per-color) | — | `Get-Service`, ไฟล์ `C:\logs\<brand>\*.container.log` |
| **GitOps deploy log** | ผล deploy ของ NSSM auto-deploy | — | `C:\logs\gitops-deploy.plain.log` |

### ทำไมตู้ถึงไม่ขึ้น Kibana

Release build **ไม่ plant `DebugTree`** — บรรทัด `Timber.d("[Serial] …")` ทั่วไปจึงไม่ขึ้น
`adb logcat` ใน release. แต่มี **2 ช่องทางที่ยังเก็บ log ได้ใน release**:

1. **ไฟล์ log บนตู้ (durable)** — `FileLoggingTree` ถูก plant ทั้ง debug และ release
   (`feat(logging): plant FileLoggingTree in both builds`) → Timber **ทุกบรรทัด** (ทุก tag) ถูก
   เขียนลงไฟล์รายวัน `log-YYYY-MM-DD.txt` ที่
   `/sdcard/Android/data/com.vendingmachine.app/files/logs/` เก็บไว้ 14 วัน
   (`LogRetention`) ดึงออกด้วย `adb pull` ได้แม้ release (ไม่ต้อง `run-as`).
2. **tag ที่ echo ลง logcat ได้แม้ release** — `FileLoggingTree(echoToLogcat = true)` ใน release
   ส่งบรรทัดต่อไปนี้ลง `android.util.Log` ด้วย จึง `adb logcat | grep <tag>` ได้สด:
   - **`VMC_SERIAL`** — serial trace ที่ `SerialService.kt` ใช้ `Timber.tag("VMC_SERIAL")`
   - **`FOOTER_LOGO`** — log การโหลดรูป footer ใน `Footer.kt` (เพิ่มภายหลัง commit 68e8170)

server log ฝั่ง API ยังอยู่ใน Kibana ตามเดิม — แต่ log ของตู้ (dispense/serial/footer) **ไม่มีทาง
อยู่ใน Kibana**; ดูได้จาก `adb logcat` (สด) หรือไฟล์ log บนตู้ (ย้อนหลัง 14 วัน) เท่านั้น.

---

## 2. วิธีเข้าถึงแต่ละแหล่ง

### 2.1 ตู้ (kiosk) — `adb logcat`

```bash
# ต่อตู้ผ่าน WiFi / Tailscale (พอร์ต ADB = 5555)
adb connect <ip>:5555
adb devices

# serial / dispense trace (ตัวสำคัญสุดเวลา "กดเบิกแล้ว locker ไม่เปิด")
adb logcat | grep VMC_SERIAL
# หรือ filter แบบ tag-only
adb logcat -s VMC_SERIAL

# footer logo
adb logcat -s FOOTER_LOGO

# diagnostics แบบ read-only (ส่ง 0x53/0x01 เช็คสถานะ ไม่จ่ายของจริง)
adb logcat -s VMC_TEST VMC_SERIAL

# log ทั้งแอป (เฉพาะ debug build เท่านั้นที่เห็น Timber tag ทั้งหมด)
adb logcat --pid=$(adb shell pidof com.vendingmachine.app)

# เคลียร์ buffer ก่อนเริ่ม reproduce
adb logcat -c
```

Package = `com.vendingmachine.app`, activity = `.MainActivity`.
คำสั่ง ADB เพิ่มเติมดูที่ `vending-machine-kotlin/ADB-HELPER.md`.

**Tip:** ตัด noise ออก — exclude `okhttp` และ SELinux `avc: denied`. POLL (0x41) /
ACK (0x42) ถูก suppress ไว้ตั้งใจ; dispense จริงจะเห็น
`connect() … USE_MOCK=…` → `POLL → TX CMD` → `*** DISPENSE STATUS (0x04) …`.
ถ้าไม่เห็นบรรทัดพวกนี้ในช่วงเกิดเหตุ = ไม่มี traffic ไป VMC จริง = น่าจะรันโหมด mock.

> Release build ใช้ `run-as` ไม่ได้ → อ่าน DataStore / DB ของแอปบนตู้ production ตรง ๆ ไม่ได้.
> แต่ไฟล์ log (ข้อ 2.2) อยู่ใน app-private external storage จึง `adb pull` ได้แม้ release.

### 2.2 ไฟล์ log บนตู้ (durable, ย้อนหลัง 14 วัน)

ใช้เมื่อเหตุเกิดไปแล้วและ `adb logcat` buffer หมุนทับไปแล้ว — Timber ทุกบรรทัดถูกเขียนลงไฟล์รายวัน:

```bash
# ดึงไฟล์ log ทั้งโฟลเดอร์ออกมา (ทำงานทั้ง debug/release ไม่ต้อง run-as)
adb pull /sdcard/Android/data/com.vendingmachine.app/files/logs/ ./kiosk-logs

# ไฟล์ชื่อ log-YYYY-MM-DD.txt — เปิดไฟล์ของวันที่เกิดเหตุ แล้ว grep หา VMC_SERIAL / FOOTER_LOGO
grep VMC_SERIAL ./kiosk-logs/logs/log-2026-06-14.txt
```

- รูปแบบบรรทัด: `HH:MM:SS.mmm  D/<TAG>: <message>` (เวลา UTC, ดู `LogFormat`).
- เก็บไว้ **14 วัน** แล้ว prune อัตโนมัติ (`LogRetention`, ไฟล์เกินหน้าต่างจะถูกลบ).
- `LogFileManager` เขียนแบบ background + flush ตอนแอปเข้า background.

### 2.3 Kibana / ELK (server log — internal only)

- Kibana UI เป็น **internal** (ไม่เปิด public). index ที่ใช้ค้น = `logs-vending-machine-*`.
- เครื่อง ELK และ endpoint รับ log อยู่หลัง infra ภายใน — **อย่าใส่ IP/hostname จริง
  ลงเอกสารที่ส่งลูกค้า** (ELK box / `service.name` เป็นข้อมูล internal). ถามทีม infra
  เพื่อขอ URL Kibana + บัญชีเข้าใช้.
- ค้นด้วย field `service.name:<brand>-vending-machine-api` แล้วเช็ค `@timestamp` ล่าสุด
  ว่าสด (ควรเป็นวินาทีล่าสุด) เพื่อยืนยันว่า log ยังไหลเข้าอยู่.

> API log มาจาก NestJS 11 + Fastify + **Pino** (`nestjs-pino`) → stdout ของ container
> → NSSM log-tail → Logstash `/logs-intake` → Elasticsearch → Kibana.

### 2.4 Container log บนเซิร์ฟเวอร์ (Windows / Docker)

```powershell
# สถานะ container ทุกตัว
docker compose ps

# tail log ของ API
docker compose logs -f api

# ดูว่า active color ตอนนี้คืออะไร (blue/green)
docker ps --filter name=<brand>

# deploy log ของ NSSM auto-deploy
Get-Content C:\logs\gitops-deploy.plain.log -Tail 100 -Wait
Get-Content C:\logs\gitops-deploy.stderr.log -Tail 100 -Wait
```

### 2.5 NSSM log-tail services (ท่อส่ง container log → Kibana)

แต่ละ brand มี tail service **2 ตัว** (ตาม blue-green): `...-log-tail-blue` และ
`...-log-tail-green` บวก HTTP shipper. ทั้งหมดต้อง `Running`:

```powershell
# เช็คว่าครบและรันอยู่
Get-Service *-log-tail-blue, *-log-tail-green, *-log-http-shipper

# เช็คไฟล์ปลายทางว่ายัง write อยู่ (LastWriteTime ต้องสด)
Get-ChildItem C:\logs\<brand>\*.container.log
```

ท่อ: `watch-docker-log.ps1` (`docker logs -f <container>` → `C:\logs\<brand>\<svc>.<color>.container.log`)
→ `ship-log-http.ps1` → Logstash HTTP `/logs-intake`.
รายละเอียดติดตั้งดูที่ `vending-machine-gitops/observability/README.md`.

### 2.6 Mailpit (outbound mail)

- API ส่ง mail ผ่าน SMTP `1025`, ดูเมลที่ออกได้ที่ Mailpit UI พอร์ต `8025`.
- Local dev: `docker compose up` ของ `vending-machine-api` ยก Mailpit ขึ้นมาให้แล้ว.
- บนเซิร์ฟเวอร์ Windows: Mailpit ติดตั้งเป็น Windows Service ผ่าน NSSM
  (`setup-windows-services.ps1`). ถ้า "mail ไม่ส่ง" ให้เปิด Mailpit UI ดูก่อนว่ามาถึง
  Mailpit ไหม — ถ้ามา = ฝั่ง API ส่งสำเร็จ, ปัญหาอยู่ปลายทาง config.

---

## 3. Symptom → ดูที่ไหน (Playbook)

| อาการ | ดูที่ไหนก่อน | คำสั่ง / ขั้นตอน | สาเหตุที่พบบ่อย |
|-------|--------------|------------------|------------------|
| กดเบิกแล้ว locker ไม่เปิด / สินค้าไม่ออก | **`VMC_SERIAL`** (ตู้) | `adb logcat \| grep VMC_SERIAL` ในช่วงเกิดเหตุ | ไม่เห็น `DISPENSE STATUS (0x04)` = ไม่มี traffic จริง → สงสัย `USE_MOCK` หรือ serial ไม่ connect |
| เบิก 1 ชิ้น แต่ stock/pickup ถูกหัก 2 | **`VMC_SERIAL`** (ตู้) | ดูว่ามี `DISPENSE STATUS (0x04) status=0x02` (สำเร็จ) ซ้ำหลายเฟรมต่อชิ้นเดียว | firmware retransmit เฟรม success — มี idempotency guard กันรายงานซ้ำแล้ว (commit cbe0773); ถ้าเห็นหักซ้ำ = guard หลุด |
| Footer logo หาย | **`FOOTER_LOGO`** (ตู้) + SQL ฝั่ง API | `adb logcat -s FOOTER_LOGO` → ถ้าเห็น `logo load FAIL url=…` = Coil โหลดรูปพลาด; ถ้าไม่ render เลย = server ส่ง url ว่าง | ส่วนใหญ่ **transient** ไม่ใช่ data bug — เช็ค `machine_themes.company_image_id` (ดู SQL §4) |
| brand หนึ่งหยุดส่ง log เข้า Kibana (อีก brand ยังไหล) | **NSSM log-tail** (เซิร์ฟเวอร์) | `Get-Service *-log-tail-blue,*-log-tail-green,*-log-http-shipper` + เช็ค active color | deploy flip color แต่ tail service ของ color ใหม่ไม่มี/ตั้งชื่อผิด → log ถูกตัดเงียบ ๆ (ดู §4) |
| mail ไม่ส่ง | **Mailpit** | เปิด Mailpit UI พอร์ต `8025` | ถ้ามาถึง Mailpit = API ส่ง OK; ถ้าไม่มา = config SMTP / use case ฝั่ง API |
| API error / 4xx-5xx, request พลาด | **Kibana** | ค้น `service.name:<brand>-vending-machine-api` + ช่วงเวลา | ดู Pino log ของ request นั้น |
| stock ไม่ตรง / picked_quantity เพี้ยน | **Kibana** (API sync) + DB | เทียบ log sync กับค่าใน DB | sync delete-all+insert ทับค่า local (ใช้ snapshot + `maxOf`) |
| socket หลุด / real-time ไม่อัปเดต | ตู้ (tag `Socket`) | debug build: `adb logcat … \| grep -i socket` | Socket.IO reconnect / api-key |
| ตู้ขึ้น offline ใน Manage แต่ Monitor ยัง online | API `/heartbeat` vs socket | เทียบ Monitor (socket) กับ `lastHeartbeatAt` 90s | heartbeat หยุดส่งแต่ socket ยังอยู่ — api-key ยัง OK |

---

## 4. Diagnostic SQL / Commands ที่ใช้บ่อย

### Footer logo — เช็ค data ฝั่ง server

```sql
SELECT m.code,
       mt.company_image_id,
       CASE WHEN mt.company_image_id IS NULL THEN 'NO_LOGO_ID'
            WHEN f.id IS NULL THEN 'FILE_MISSING'
            ELSE 'OK' END AS logo_state
FROM machine_themes mt
JOIN machines m ON m.id = mt.machine_id
LEFT JOIN files f ON f.id = mt.company_image_id
WHERE m.code IN (...);
-- logo URL = {baseUrl}/uploads/{files.path}/{files.filename}
```

`logo_state = OK` ทั้งคู่แต่ลูกค้าบอกหาย → เกือบทุกครั้งเป็น **transient** (jang push
ไม่ทันหรือ Coil พลาดชั่วคราว) ไม่ใช่ data bug.

### Brand log-tail gap — fix

```powershell
# รันเป็น Admin, เรียก pwsh ด้วย full path (ไม่มีใน PATH บนบางเครื่อง)
& "C:\Program Files\PowerShell\7\pwsh.exe" `
  -File C:\gitops\setup-observability-services.ps1 `
  -BlueContainerName  <prefix>-blue-vending-machine-api-1 `
  -GreenContainerName <prefix>-green-vending-machine-api-1
```

ต้อง pass `-BlueContainerName` / `-GreenContainerName` ให้ครบทั้งสอง color — default ของ
สคริปต์ derive ชื่อ container ผิด → tail ผูกกับ container ที่ไม่มีจริง → จับ log ไม่ได้เงียบ ๆ.
หลังแก้ ยืนยันใน Kibana ว่า `@timestamp` ล่าสุดของ `service.name:<brand>-vending-machine-api`
สดเป็นวินาที.

---

## 5. Common Gotchas

- **`SerialService.USE_MOCK` ต้องเป็น guarded property เสมอ** — ปัจจุบันคือ
  `_useMock = if (BuildConfig.DEBUG) value else false` (release ล็อกฮาร์ดแวร์,
  debug toggle ได้). **ห้ามลดเหลือ `var = true/false` เด็ดขาด** — เคยหลุด mock mode
  ขึ้นทั้ง fleet มาแล้ว (ทุกเครื่อง fake dispense สำเร็จแต่ locker ไม่เปิด). ถ้าเห็นมัน
  กลายเป็น plain var เมื่อไหร่ นั่นคือบั๊ก.
- **อย่าหา serial/footer log ใน Kibana** — ไม่มี. ตู้ = `adb logcat` (สด) หรือไฟล์ log บนตู้ (ย้อนหลัง).
- **Release ไม่ plant DebugTree** — debug log ส่วนใหญ่ไม่ขึ้น `adb logcat` ใน release; เหลือแต่ tag ที่
  echo ไว้ (`VMC_SERIAL`, `FOOTER_LOGO`). แต่ **ทุกบรรทัดยังถูกเขียนลงไฟล์ log บนตู้** (`FileLoggingTree`
  plant ทั้งสอง build) — เหตุที่ logcat buffer หมุนทับไปแล้วยัง `adb pull` ไฟล์ย้อนหลัง 14 วันได้.
- **Release build ใช้ `run-as` ไม่ได้** — อ่าน DB/DataStore บนตู้ production ตรง ๆ ไม่ได้;
  แต่ไฟล์ log อยู่ใน app-private external storage จึง `adb pull` ได้.
- **blue-green มี tail service 2 ตัวต่อ brand** — `deploy.ps1` flip color แต่ไม่ reinstall/
  verify observability → color ที่ไม่ active เป็นจุดบอด. ตรวจ `Get-Service` ทุกครั้งหลัง deploy.
- **`adb logcat -c` ก่อน reproduce** — เคลียร์ buffer เก่าออก จะอ่านง่ายขึ้น.
- **VMC_TEST = read-only diagnostics** — ส่ง 0x53 / 0x01 เช็คสถานะ ไม่จ่ายของจริง ใช้ทดสอบ
  serial ได้โดยไม่เสียสินค้า.

---

## เอกสารอ้างอิง

- แผนอบรม internal — Session 3 ใช้เอกสารนี้ (ดูชุดเอกสาร dev)
- ใน repo vending-machine-kotlin: `ADB-HELPER.md` — คำสั่ง ADB ครบชุด
- ใน repo vending-machine-kotlin: `AGENTS.md` — "Serial Mock & Logging", USE_MOCK guard
- ใน repo vending-machine-kotlin: `util/logging/` — `FileLoggingTree`, `LogFileManager`, `LogFormat`, `LogRetention` (durable file log)
- `vending-machine-gitops/observability/README.md` — ติดตั้ง log-tail / shipper / blue-green
- `vending-machine-elk/README.md` — ELK stack (Elasticsearch / Kibana / Logstash)
- `vending-machine-api/README.md` — Mailpit (`8025`/`1025`), Docker services
