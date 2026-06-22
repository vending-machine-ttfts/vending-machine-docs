# 07 — Offline Deploy & Update Cheat-Sheet (web / api / kotlin)

หน้าเดียวรวม flow **อัปเดตเวอร์ชันใหม่** ของ 3 แอป เมื่อ server/ตู้ offline. คำสั่งย่อมาจาก source จริง — รายละเอียดเต็มดูลิงก์ท้ายแต่ละหัวข้อ.

> source of truth (อย่าก๊อปคำสั่งจากที่อื่น):
> - web/api deploy loop + blue-green: `vending-machine-gitops/deploy.ps1`
> - web/api update prod (stop→pull→start): `vending-machine-gitops/RUNBOOK-update-prod-sites.md`
> - web/api air-gap delta: `tech-team-handbook.md` §1.4
> - kotlin OTA: `04-android-release-ota.md`

> **ดูแบบ visual:** [`../system-overview.html`](../system-overview.html) แท็บ *Offline Deploy (air-gap D)* (animated ทีละขั้น) · [`../system-flow-deck.html`](../system-flow-deck.html) สไลด์ 14

**TL;DR เลือกวิธี:** เปิดเน็ตชั่วคราวได้ → **A** (ง่ายสุด) · ห้ามเปิด public แต่มี Tailscale → **B** · มีเครื่องกลางในไซต์ → **C** · air-gap เข้ม → **D** (§1–§2). core เหมือนกันหมด — เอา image+code เข้าเครื่อง แล้ว `deploy.ps1` flip blue-green เอง.

---

## 0. บริบท — "offline" มี 2 ระดับ (อ่านก่อน)

| ระดับ | เคส | image / code เข้าเครื่องยังไง |
|---|---|---|
| **A. ตัดเน็ตหลังส่งมอบ** (เคสปกติ เช่น Denso) | server **มีเน็ตตอน setup** → `docker compose pull` + `git pull` ได้ปกติ. หลังส่งมอบตัดเน็ต → `deploy.ps1` แค่ idle (pull fail, container เดิมรันต่อ) | deploy ใหม่ = ทีม remote เข้าผ่าน **Tailscale** แล้วเสิร์ฟ git/image เข้า LAN/tailnet หรือใช้ USB (ระดับ B) |
| **B. Air-gap จริง** (ไม่มีเน็ตเลย) | ดึงจาก GitHub/Docker Hub ไม่ได้เลย | **image** → USB `docker save`/`load`; **code** → git mirror ใน LAN หรือ copy ไฟล์ตรง |

- **Tailscale = ช่องทาง remote ของทีม** ไม่ใช่เน็ตออกของ server. เป็นเส้นเดียวที่ push อัปเดตให้ลูกค้า offline ได้ → ต้องมีเสมอ.
- ของ **native บน host** (SQL Server / Redis / Mailpit) ไม่ใช่ container — ไม่เกี่ยวกับ deploy นี้.

### 0.1 เลือกวิธีส่ง build เข้าไซต์ — 4 ทาง (ตามนโยบายลูกค้า)

| วิธี | เน็ต server | เหมาะเมื่อ | how | ข้อควรระวัง |
|---|---|---|---|---|
| **A. เปิดเน็ตชั่วคราว** (maintenance window) | เปิดเป็นช่วง | ลูกค้าอนุญาตเปิดเน็ตชั่วคราว | เปิดเน็ต → `deploy.ps1` git pull + `docker compose pull` เองอัตโนมัติ → ปิดเน็ต | **ง่าย/เร็วสุด** — แนะนำถ้าทำได้ |
| **B. Tailscale exit node** | ออกผ่าน tailnet (ไม่เปิด public) | อยากคุม scope | ทีมตั้ง exit node / subnet-router ในtailnet → server route ออกช่วง deploy → pull ปกติ | ต้อง config exit node + ACL; throughput ผ่านเครื่องทีม |
| **C. Mirror ใน LAN** | ไม่ต้องออกเน็ตเลย | ไซต์มีเครื่อง programmer/เครื่องกลาง | git remote (+ option registry) บนเครื่องในไซต์ → `deploy.ps1` pull จาก LAN | ตั้งครั้งแรกยุ่ง; Windows registry มี caveat |
| **D. USB 100% offline** | ไม่มีเลย | air-gap เข้ม | `docker save`/`load` + git bundle/copy (§1 + §2) | manual ทุกครั้ง; ต้องเครื่อง **Windows** ทำ `docker save` |

- ⚠️ **Tailscale เปล่าๆ pull จาก Docker Hub/GitHub ไม่ได้** — tailnet ไม่ใช่เน็ตออก. ต้อง exit node (B) หรือ mirror (C).
- **kotlin/APK ไม่ติดข้อจำกัดนี้** — APK host บน API ของ site เอง → upload ผ่าน LAN/Tailscale แล้ว kiosk pull ในวง (หรือ adb USB ตรง §3d). ใช้ได้กับทุกวิธี A–D.
- §1 + §2 ข้างล่างคือรายละเอียดของวิธี **D**; A/B/C ใช้ flow ปกติ (`deploy.ps1` pull เอง) แค่ต่างที่ "เน็ตมาจากไหน".

---

## 1. web/api — deploy image ใหม่ (air-gap, ระดับ B)

image build จาก CI (`docker push sangzn34/vending-machine-{api,web}:<tag>`, tag = `<semver>-<shorthash>` เช่น `0.0.3-0f2f9f6`). ไม่ต้อง build เอง — CI build ให้แล้ว แค่ pull ลงมา save. air-gap ก็ขนผ่าน `.tar`.

> ⚠️ image เป็น **Windows container** (`nanoserver:ltsc2025`) → ขั้น `docker pull`/`docker save` **ต้องทำบนเครื่อง Windows ที่ Docker = Windows-container mode**. **macOS/Linux ทำไม่ได้** (Docker Desktop = Linux VM). prod ต้องเป็น **Server 2025** จึง `docker run` ได้ (kernel match).

**1a. ฝั่ง build (เครื่อง Windows + มีเน็ต) — pull จาก CI แล้ว export เป็น tar**
```powershell
# tag ต้องตรงกับที่ docker-compose.template.yml อ้าง (ดู image: ในไฟล์นั้น)
docker pull sangzn34/vending-machine-api:<tag>
docker pull sangzn34/vending-machine-web:<tag>
docker save sangzn34/vending-machine-api:<tag> -o vending-machine-api.tar
docker save sangzn34/vending-machine-web:<tag> -o vending-machine-web.tar
# ขน .tar ผ่าน USB / Tailscale file copy ไปเครื่อง prod
```

**1b. ฝั่ง prod (offline) — load + ชี้ compose ให้ตรง tag**
```powershell
docker load -i vending-machine-api.tar
docker load -i vending-machine-web.tar
docker images   # ยืนยันเห็น sangzn34/vending-machine-{api,web}:<tag> ตรง
# แก้ docker-compose.template.yml (ใน $GitOps) ให้ image: ชี้ <tag> ที่เพิ่ง load
```

**1c. trigger deploy (blue-green flip)** — ดู §2 (เหมือนกัน). `deploy.ps1` เห็น compose-hash เปลี่ยน → start slot สำรองจาก image ใหม่ → health check → flip IIS (slot เก่าค้างไว้ถ้า slot ใหม่ fail = ปลอดภัย).

→ เต็ม: `tech-team-handbook.md` §1.4

---

## 2. web/api — apply code/config ใหม่ (gitops files)

flow มาตรฐาน per brand. **ลำดับสำคัญ:** stop → pull → start (service โหลด `deploy.ps1` ใหม่ตอน restart เท่านั้น).

```powershell
$nssm    = "C:\nssm\win64\nssm.exe"
$Brand   = "ttfts"                  # หรือ stm / denso
$GitOps  = "C:\gitops-$Brand"
$Service = "$Brand-gitops-deploy"

& $nssm stop $Service; Start-Sleep 3
# pull จาก origin (online) — air-gap: pull จาก git mirror ใน LAN หรือ copy ไฟล์ทับแทนบรรทัดนี้
git -c "safe.directory=$($GitOps -replace '\\','/')" -C $GitOps pull
& $nssm start $Service
```

**บังคับ redeploy ทันที** (ถ้า image tag เดิมแต่อยาก re-render env / apply ตอนนี้เลย):
```powershell
& $nssm stop $Service; Start-Sleep 3
Remove-Item "$GitOps\.last-compose-hash" -ErrorAction SilentlyContinue
& $nssm start $Service
```

→ เต็ม (รวม pin host vars, per-brand checklist): `vending-machine-gitops/RUNBOOK-update-prod-sites.md`

---

## 3. kotlin — OTA APK (เครื่องตู้)

ไม่ต้อง copy ไฟล์ขึ้น server เอง — ปล่อยผ่าน **Web admin** แล้ว kiosk pull เอง. ใช้ได้แม้ site offline (APK host บน API ของ site เดียวกัน, kiosk โหลดผ่าน LAN/Tailscale).

**3a. build + sign** (จาก root `vending-machine-kotlin`)
```bash
./gradlew assembleRelease     # Windows: gradlew.bat assembleRelease
# → app/build/outputs/apk/release/app-release.apk (เซ็น release key อัตโนมัติ)
```
> ⚠️ **ต้อง bump `versionCode` (+1) ทุกครั้ง** ใน `app/build.gradle.kts` — API เทียบ `versionCode` (integer) เท่านั้น ไม่งั้น kiosk มองไม่เห็นเวอร์ชันใหม่.
> ⚠️ เซ็นด้วย **key เดิมเสมอ** — signature เปลี่ยน = ติดตั้งทับไม่ได้ (ต้อง uninstall → เสีย Device Owner).

**3b. publish** — Web admin → Settings → App Versions (`/settings/app-versions`) → upload `app-release.apk`, เลือก `appName` = `vending-machine` (หรือ `vending-updater` ตอน self-update). `isActive=true`.

**3c. rollout** — auto: kiosk เช็คตอนเปิดแอป + ทุก 5 นาที (เฉพาะ idle ไม่มีคน login) → pull เอง. ไม่มี per-machine targeting (all-or-nothing ต่อ `appName`).

**3d. manual trigger** (ทดสอบ/ด่วน ผ่าน adb):
```bash
adb shell am start -a android.intent.action.VIEW \
  -d 'vending-ttfts-updater://update?apkUrl=https%3A%2F%2F<host>%2Fuploads%2Fapk%2F<file>.apk'
# self-update ตัว updater: ต่อท้าย &selfUpdate=true (apkUrl ต้อง URL-encoded)
```

→ เต็ม (signing, deep-link scheme, rollback): `04-android-release-ota.md`

---

## 4. Verify (หลัง deploy ทั้ง 3)

```powershell
# web/api
Get-Content "C:\logs\$Brand\$Service.plain.log" -Tail 30          # เห็น render + flip สำเร็จ
& "C:\Program Files\PowerShell\7\pwsh.exe" -File "$GitOps\doctor.ps1" -Brand $Brand -GitOpsDir $GitOps
Invoke-WebRequest -UseBasicParsing http://localhost:<BLUE_API_PORT>/api/v1/health   # {"ok":true}
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"
```
```bash
# kotlin (ตู้)
adb shell dumpsys package com.vendingmachine.app | grep versionCode   # ตรง versionCode ที่ bump
```
PASS: `doctor.ps1` RESULT PASS · `/api/v1/health` ตอบ `{"ok":true}` · kiosk versionCode = ค่าใหม่.

---

## 5. Rollback

| แอป | วิธี |
|---|---|
| **web/api (code)** | `$nssm stop` → `git reset --hard HEAD~1` (ใน `$GitOps`) → `$nssm start`. container ที่รันอยู่ไม่ถูกแตะจนกว่า redeploy รอบหน้า |
| **web/api (image)** | แก้ `docker-compose.template.yml` กลับ tag เก่า (image เก่ายังอยู่ใน `docker images`) → forced redeploy (§2) |
| **kotlin** | App Versions → toggle เวอร์ชันเก่ากลับ `isActive` (versionCode เก่าต้องสูงกว่าตัวที่จะถอย — ปกติ bump เวอร์ชัน hotfix ใหม่ดีกว่า rollback) |

---

## อ้างอิง source (verify trail)

- deploy loop / blue-green / ports: `vending-machine-gitops/deploy.ps1`
- prod update sequence: `vending-machine-gitops/RUNBOOK-update-prod-sites.md`
- air-gap delta + docker save/load: `tech-team-handbook.md` §1.4
- image build/push (CI): `vending-machine-{api,web}/.github/workflows/windows-container-build.yml`
- OTA build/publish/trigger: `04-android-release-ota.md`
- compose image tag ที่ใช้อยู่: `vending-machine-gitops/docker-compose.template.yml`
