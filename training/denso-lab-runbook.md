<!-- INTERNAL — DO NOT PUBLISH. Training answer-key. Contains live server IPs + registry handle. Keep out of customer builds. -->

# Denso Deploy Lab — Trainer Answer Key

แล็บฝึกแก้ปัญหา "run deploy denso ไม่ขึ้น" บนเครื่องทดสอบ 5 เครื่อง (brand `denso`, gitops dir `C:\gitops-denso`, ports blue 9080/9081 green 9082/9083).

## เครื่อง

| # | SSH | fault | สถานะ |
|---|---|---|---|
| test01 | `ssh administrator@149.28.150.182` | reference (api healthy) — ใช้เทียบค่าที่ถูก |
| test02 | `ssh administrator@66.42.51.45` | DB_HOST มี `\SQLEXPRESS` | api Exited |
| test03 | `ssh administrator@207.148.78.45` | DB_HOST `\SQLEXPRESS` + REDIS_DB ไม่ใช่เลข | api Restarting |
| test04 | `ssh administrator@64.176.81.35` | deploy service ถูก stop | deploy Stopped |
| test05 | `ssh administrator@139.180.213.205` | DB_HOST มี `\SQLEXPRESS` | api Restarting |

---

## สาเหตุระดับระบบ (ทุกเครื่อง) — ตัวจริงของ "deploy ไม่ขึ้น"

deploy service env `COMPOSE_PROJECT_PREFIX=Denso` (**ตัวพิมพ์ใหญ่**) → deploy.ps1 รัน `docker compose -p Denso-blue ...` → docker ปฏิเสธ:
```
invalid project name "Denso-blue": must consist only of lowercase alphanumeric...
docker compose up exit code: 1
```
→ **ทุก deploy ล้มทุก tick** ไม่มีวันขึ้น. container `denso-blue-*` ที่ยังรันอยู่เป็นของเก่า (สร้างตอน prefix ยังเป็นตัวเล็ก) ไม่ใช่ของ deploy service ปัจจุบัน — deploy service มองไม่เห็นด้วยซ้ำ.

**แก้ (systemic):** ทำให้ project prefix เป็นตัวเล็กเสมอ — แก้ใน `deploy.ps1`:
```powershell
$composeProjectPrefix = (Get-ConfigValue "COMPOSE_PROJECT_PREFIX" "").ToLower()
```
เครื่องรับผ่าน `git pull` (deploy loop pull เองทุก tick). ต้นทางควรแก้ `setup-site.ps1` ให้ lowercase brand ตอนตั้ง prefix ด้วย.

> ค่าที่ถูก vs ผิด: docker **project name** ต้อง lowercase เสมอ. ส่วน **service name** (`Denso-vending-machine-api`) เป็นตัวใหญ่ได้ ไม่เกี่ยว.

---

## สาเหตุระดับเครื่อง (api ไม่ขึ้นแม้ deploy ทำงาน)

ค่าที่ถูก (จาก test01): `DB_HOST=TEST01` · `REDIS_DB=2` · template = `mssql://sa:{{DB_PASSWORD}}@{{DB_HOST}}:1433/..` , `redis://:{{REDIS_PASSWORD}}@{{REDIS_HOST}}:6379/{{REDIS_DB}}`

**Fault A — DB_HOST มี `\SQLEXPRESS`** (test02/03/05)
แอป parse `DATABASE_URL` เป็น URL → `\` = invalid → `Error: Invalid URL` → migration ล้ม.
**DB_HOST ที่ถูก = `TEST0X` (host ล้วน)** — template เติม `:1433` ให้, SQL Express ฟัง static 1433. ไม่ใช่ `TEST0X\SQLEXPRESS:1433` (`,` `\` `:port` พังหมด) และไม่ใช่ `SQLEXPRESS` (นั่นชื่อ instance).

**Fault B — REDIS_DB ไม่ใช่ตัวเลข** (test03)
`REDIS_DB=Redis@1234` → `SELECT NaN` → `ERR invalid DB index` → crash loop. ต้องเป็นเลข index (0,1,2...).

**Fault C — deploy service stopped** (test04)
`denso-gitops-deploy` ถูก stop → ไม่มี auto-deploy.

---

## วิธีแก้ที่แนะนำ — ใช้ doctor + fix-site (0-START-HERE)

double-click `0-START-HERE.cmd` →

1. **`6) Health check`** (doctor) — ตอนนี้ชี้ตรงทุก fault:
   ```
   [FAIL] DB_HOST format - 'TEST05\SQLEXPRESS' has an instance/path separator
   [FAIL] REDIS_DB format - 'Redis@1234' is not a number
   [FAIL] compose project prefix - 'Denso' has uppercase -> invalid project name; deploy exits 1
   ```
2. **`F) Fix / redeploy`** (fix-site) — auto-fix + redeploy:
   - `1) Auto-fix DB/Redis config + redeploy` → normalize `.env.secrets` (DB_HOST host-only, REDIS_DB เป็นเลข) + force blue-green redeploy
   - `2) Redeploy blue-green only` → force deploy หนึ่งรอบ (เก็บ config เดิม)

CLI ตรงๆ:
```powershell
cd C:\gitops-denso
.\fix-site.ps1 -Brand denso -AutoFix              # แก้ config + redeploy
.\fix-site.ps1 -Brand denso -AutoFix -RedisDb 2   # ระบุ redis db index เอง
.\fix-site.ps1 -Brand denso -Redeploy             # แค่ force redeploy
```
> fix-site ไม่เรียก docker เอง — สั่งผ่าน deploy service (ลบ `.last-compose-hash` + restart) ให้ deploy.ps1 จัดการ render/ports/blue-green เอง จึงไม่หลุดจาก config จริง (ports 9080..). ต้องมี deploy.ps1 ที่ lowercase prefix แล้ว (pull ก่อน).
> test04 (deploy stopped): `Redeploy` restart service ให้เอง = หาย.

### Fallback แบบ manual (ถ้าไม่ใช้ fix-site)
```powershell
cd C:\gitops-denso
(Get-Content .env.secrets) -replace '^DB_HOST=.*','DB_HOST=TEST0X' -replace '^REDIS_DB=.*','REDIS_DB=2' | Set-Content .env.secrets
Remove-Item .last-compose-hash -Force; Restart-Service denso-gitops-deploy   # force blue-green redeploy
```
(อย่ารัน `docker compose up` มือเปล่า เว้นจำเป็น — ต้อง set `$env:UPLOAD_DATA_DIR`/`WEB_PORT`/`API_PORT` ให้ตรง 9080.. เอง และต้องใช้ project name ตัวเล็ก)

---

## วิธี re-break (รีเซ็ตให้ผู้เรียนลองใหม่)

```powershell
cd C:\gitops-denso
# Fault A
(Get-Content .env.secrets) -replace '^DB_HOST=.*','DB_HOST=TEST0X\SQLEXPRESS' | Set-Content .env.secrets
# Fault B (test03)
(Get-Content .env.secrets) -replace '^REDIS_DB=.*','REDIS_DB=Redis@1234' | Set-Content .env.secrets
# บังคับ container ให้รับค่าพังทันที (deploy ปกติจะ health-gate ไม่ยอม deploy ของพัง):
$env:WEB_PORT=9080;$env:API_PORT=9081;$env:UPLOAD_DATA_DIR='C:\gitops-denso\upload_data'
docker compose -p denso-blue -f docker-compose.generated.yml --env-file .env.api up -d --force-recreate Denso-vending-machine-api
# Fault C (test04)
Stop-Service denso-gitops-deploy
```
> health-gate: ถ้า config พัง deploy ปกติจะ fail ที่ slot สำรองแล้วคง slot ดีไว้ — re-break จึงต้อง force-recreate slot ที่ active ตรงๆ.

---

## รหัสผ่าน Redis กำหนดตอนไหน

1. ติดตั้ง Redis บน host: `setup-windows-services.ps1 -RedisPassword <pw>` → ตั้ง `requirepass`
2. setup-site: `init-secrets.ps1` เก็บ `REDIS_PASSWORD` ใน `.env.secrets` (ต้อง = ข้อ 1)
3. deploy: ประกอบ `REDIS_URL` จาก template ทุก tick
ค่าใน `.env.secrets`: `REDIS_HOST` (= DB_HOST), `REDIS_PASSWORD` (= requirepass), `REDIS_DB` (เลข index ต่อ brand). รหัสผิด → NOAUTH/WRONGPASS (คนละอาการกับ DB index ผิด).

## เมนู 1 ลง SQL ให้ไหม
**ไม่ by default.** `setup-server.ps1` section 7 ลง SQL Express เฉพาะเมื่อส่ง `-SaPassword` (เมนูไม่ส่ง → warn+skip) และต้องรันจาก **console ตรง ไม่ใช่ SSH** (DPAPI พัง). มันเช็ค SQL ที่มีอยู่ก่อนเสมอ. doctor (เมนู 6) เช็คว่ามี SQL หรือยัง (`MSSQLSERVER`/`MSSQL$*`).

## ค่าคงที่
- ports: blue 9080/9081, green 9082/9083 (จาก deploy service env `BLUE_*`/`GREEN_*`)
- active slot อ่านจาก `.active-slot`; compose service `Denso-vending-machine-api/-web`
- เทส DB login จาก host: `sqlcmd -S TEST0X -U sa -P <pw> -Q "SELECT 1"`

## ไฟล์ที่แก้ใน vending-machine-gitops (ยังไม่ commit)
- `deploy.ps1` — lowercase project prefix (systemic fix)
- `doctor.ps1` — เพิ่มเช็ค DB_HOST format / REDIS_DB format / compose project prefix
- `fix-site.ps1` — ใหม่: auto-fix config + blue-green redeploy
- `0-START-HERE.cmd` — เพิ่มเมนู `F) Fix / redeploy`
