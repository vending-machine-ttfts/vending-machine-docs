# 04 — Android Release & OTA Publish Runbook

> เอกสารอ้างอิงสำหรับนักพัฒนา (developer reference) — วิธี build / sign release APK ของแอป kiosk
> และปล่อยอัปเดตขึ้นเครื่องลูกค้าแบบ OTA (silent install) ผ่าน `vending-updater`
>
> เอกสารนี้รองรับ **Session 4** ในแผนอบรม internal (Deploy/OTA + เพิ่มลูกค้า/เครื่องใหม่)
> และต่อเนื่องจาก [03-new-site-onboarding.md](03-new-site-onboarding.md) (white-label add site)
>
> ทุกข้อมูลถูก verify จาก source จริง: `vending-machine-kotlin/app/build.gradle.kts` + `gradle.properties`,
> `vending-updater` (README + `UpdateActivity.java` + `AndroidManifest.xml`),
> `vending-machine-api` (`app-version` module + `app-update.controller.ts`),
> `vending-machine-web` (`settings/app-versions.tsx`), และ `AppUpdater.kt` ฝั่ง kiosk
> จุดที่ยังยืนยันไม่ได้จะใส่ `⚠️ verify` กำกับไว้

---

## 1. ภาพรวม OTA Flow

ระบบ OTA มี 4 ส่วนทำงานร่วมกัน:

- **Build** — `vending-machine-kotlin` → release APK (signed)
- **Host** — อัปโหลด APK เข้า **Web admin** → API เก็บไฟล์ไว้ใต้ `/uploads/apk/` + บันทึก record (`appName`, `versionCode`)
- **Check** — kiosk เรียก API `GET /api/v1/machine/app-update/check` เป็นระยะ → ถ้ามีเวอร์ชันใหม่ได้ `apkUrl` กลับมา
- **Install** — kiosk ยิง deep link ไปที่ `vending-updater` → updater ดาวน์โหลด APK + ติดตั้งเงียบ (Device Owner)

```text
  +-------------------+        ./gradlew assembleRelease
  | vending-machine-  |  ----------------------------------> app-release.apk (signed)
  |   kotlin (dev)    |
  +-------------------+
            |  upload via Web admin (Settings > App Versions)
            v
  +-------------------+        POST /api/v1/portal/app-versions (multipart: file)
  |  Web admin (web)  |  ----------------------------------> +------------------+
  +-------------------+                                       | API (NestJS)     |
                                                              | saves APK ->     |
                                                              | /uploads/apk/    |
                                                              | row: appName,    |
                                                              | versionCode,     |
                                                              | isActive=true    |
                                                              +--------+---------+
                                                                       ^
   kiosk (vending-machine-kotlin, on machine)                          |
  +-------------------------------------------+   GET .../app-update/check?
  | AppUpdater.checkOnStart() / periodic 5min | --appName=...&versionCode=N
  |   if hasUpdate:                           | <-- { hasUpdate, apkUrl, ... } -+
  |   launchUpdater(apkUrl, selfUpdate)       |
  +---------------------+---------------------+
                        |  Intent ACTION_VIEW
                        |  vending-ttfts-updater://update?apkUrl=<encoded>[&selfUpdate=true]
                        v
  +-------------------------------------------+
  | vending-updater (UpdateActivity)          |
  |   1. DownloadManager -> update.apk        |
  |   2. if Device Owner -> PackageInstaller  |  (silent, no user prompt)
  |      else (root)     -> su -c pm install  |  (fallback)
  |   3. relaunch vending app                 |
  +-------------------------------------------+
```

หมายเหตุ: `vending-updater` เป็นแอปแยกคนละ package (`com.ttfts.vendingupdater`) ที่ติดตั้งไว้บนเครื่อง
เครื่องส่งมอบจะตั้ง updater เป็น **Device Owner** ครั้งเดียวตอน provision (ดู
[03-new-site-onboarding.md](03-new-site-onboarding.md) + `vending-updater/README.md`)

---

## 2. Build & Sign Release APK

### 2.1 คำสั่ง build (จาก root ของ `vending-machine-kotlin`)

```bash
# macOS / Linux
./gradlew assembleRelease

# Windows
gradlew.bat assembleRelease
```

ไฟล์ที่ได้: `app/build/outputs/apk/release/app-release.apk` (signed ด้วย release key อัตโนมัติ)

### 2.2 Signing config — มาจากไหน

`app/build.gradle.kts` ประกาศ `signingConfigs.release` แล้ว `buildTypes.release` ผูก
`signingConfig = signingConfigs.getByName("release")` ให้ `assembleRelease` เซ็นให้เลย ค่าทั้งหมดอ่านจาก
**Gradle properties** (ไม่ได้ hardcode):

```kotlin
// app/build.gradle.kts
signingConfigs {
    create("release") {
        storeFile = file("${rootProject.projectDir}/${project.property("VENDING_STORE_FILE")}")
        storePassword = project.property("VENDING_STORE_PASSWORD") as String
        keyAlias = project.property("VENDING_KEY_ALIAS") as String
        keyPassword = project.property("VENDING_KEY_PASSWORD") as String
    }
}
```

ค่าจริงปัจจุบันตั้งไว้ใน `gradle.properties` (committed):

```properties
VENDING_STORE_FILE=vending-release.keystore
VENDING_STORE_PASSWORD=...   # อยู่ใน gradle.properties
VENDING_KEY_ALIAS=...
VENDING_KEY_PASSWORD=...
```

- **Keystore:** `vending-release.keystore` วางอยู่ที่ root ของ repo (`rootProject.projectDir`) และ committed ใน git
- ทุก release **ต้องเซ็นด้วย key เดิม** — ถ้า signature เปลี่ยน เครื่องจะติดตั้งทับไม่ได้
  (ต้อง uninstall ก่อน → เสีย Device Owner / data ทั้งหมด) ดู `vending-machine-kotlin/KIOSK_SETUP.md`
- ⚠️ verify: production keystore/password ฝังอยู่ใน `gradle.properties` ที่ committed — ทีมควรพิจารณาย้ายไป
  `local.properties` (ถูก gitignore แล้ว) หรือ CI secret ก่อนเปิด repo ออกนอกทีม

### 2.3 Bump เวอร์ชัน

แก้ที่ `app/build.gradle.kts` → `defaultConfig` (ปัจจุบัน `versionCode = 49`, `versionName = "0.0.43"`):

```kotlin
defaultConfig {
    versionCode = 50          // <- เพิ่มทีละ 1 เสมอ (เป็น integer)
    versionName = "0.0.44"    // <- semantic version ที่โชว์ให้คน
}
```

กฎสำคัญ: ฝั่ง API เทียบ **`versionCode` (integer) เท่านั้น** ในการตัดสินว่ามีอัปเดต
(`CheckAppUpdateUseCase`: `latest.versionCode <= currentVersionCode` → ไม่มีอัปเดต) ดังนั้น
**ต้องเพิ่ม `versionCode` ทุกครั้ง** ไม่งั้น kiosk จะมองไม่เห็นเวอร์ชันใหม่

---

## 3. Publish / Host APK

ปล่อยผ่าน **Web admin** ไม่ต้อง copy ไฟล์ขึ้น server เอง:

1. เข้า Web admin → **Settings → App Versions** (`/settings/app-versions`)
2. กด upload เลือกไฟล์ `app-release.apk`
   - ฟอร์มมี `appName` ให้เลือก 2 ค่า: **`vending-machine`** หรือ **`vending-updater`**
     (จาก `app-versions.tsx` → `appNameOptions`)
   - `versionName` / `versionCode` ระบบ parse จาก APK manifest ให้อัตโนมัติ (`parseApkManifest`)
     — เช็คให้ตรงกับที่ bump ใน gradle
   - ใส่ `releaseNotes` ได้
3. กด submit → API (`POST /api/v1/portal/app-versions`, multipart field `file`):
   - เซฟไฟล์ลง `<uploadDir>/apk/<appName>-v<versionName>-<uuid>.apk`
   - สร้าง record พร้อม `isActive = true`
   - ถ้า `versionCode` ซ้ำกับที่มีอยู่ของ `appName` เดิม → API ตอบ `400` (กัน duplicate)

APK ถูก host แบบ static ที่:

```text
<baseUrl>/uploads/apk/<filename>.apk
```

`<baseUrl>` มาจาก `ApplicationConfigService.baseUrl` ของแต่ละ site ⚠️ verify: ต้องเป็น URL ที่
**เครื่อง kiosk เข้าถึงได้** (LAN/Tailscale ของ site นั้น) — updater ดาวน์โหลดตรงจาก URL นี้

> **Deep-link URI ที่ updater รอรับ** (quote ตรงจาก source):
>
> ```text
> vending-ttfts-updater://update?apkUrl=<encoded_url>
> vending-ttfts-updater://update?apkUrl=<encoded_url>&selfUpdate=true
> ```
>
> - scheme: `vending-ttfts-updater` (manifest `<data android:scheme="vending-ttfts-updater" />`,
>   intent-filter ไม่ผูก host — ต้องมี `apkUrl` เป็น query param)
> - `apkUrl` ต้อง **URL-encoded** (ฝั่ง kiosk ใช้ `Uri.encode(apkUrl)` ก่อนต่อ string)
> - `selfUpdate=true` = ติดตั้งทับตัว updater เอง (ใช้ตอนอัปเกรด `vending-updater`)

---

## 4. Trigger + Rollout

### 4.1 Auto-rollout (production path)

หลัง upload + `isActive=true` แล้ว **ไม่ต้องสั่งอะไรเพิ่ม** — kiosk เป็นฝ่าย pull เอง:

- `AppUpdater.checkOnStart()` รันตอน app เปิด (มี timeout 5 วิ)
- `AppUpdater.startPeriodicCheck()` เช็คทุก **5 นาที** แต่ **เฉพาะตอนไม่มีคน login** (idle screen) เท่านั้น
- เช็คทั้ง `vending-machine` และ `vending-updater` (self-update) ด้วย `versionCode` ที่ติดตั้งอยู่จริง
- ถ้า `hasUpdate && apkUrl != null` → เรียก `launchUpdater(apkUrl, selfUpdate)` ยิง deep link ให้ updater ทันที
- ก่อนยิง deep link (เฉพาะ vending-machine self-update) kiosk emit `notifyGoingDown(app_update)` ผ่าน socket
  เพื่อให้ monitor history แท็ก disconnect รอบนี้เป็น `event_source=app_update` (ไม่นับเป็น offline จริง)

> **rollout เป็น all-or-nothing ต่อ `appName`:** record ที่ `isActive=true` + `versionCode` สูงสุดคือเวอร์ชัน
> ที่ทุกเครื่องของ site นั้นจะดึง ไม่มี per-machine targeting (`CheckAppUpdateUseCase` query แค่ `appName`+`isActive`)
> ถ้าต้องการ rollback ให้ toggle เวอร์ชันเก่ากลับมา active (ดูข้อ 5)

### 4.2 Manual trigger (ทดสอบ / push ด่วน) ผ่าน adb

ยิง deep link ตรงไปที่ updater บนเครื่อง (จาก README ของ `vending-updater`):

```bash
adb shell am start \
  -a android.intent.action.VIEW \
  -d 'vending-ttfts-updater://update?apkUrl=https%3A%2F%2F<host>%2Fuploads%2Fapk%2F<file>.apk'
```

self-update ตัว updater:

```bash
adb shell am start \
  -a android.intent.action.VIEW \
  -d 'vending-ttfts-updater://update?apkUrl=https%3A%2F%2F<host>%2Fuploads%2Fapk%2F<file>.apk&selfUpdate=true'
```

### 4.3 Device Owner requirement

silent install (ไม่มี popup ให้กดยืนยัน) ทำได้ **ก็ต่อเมื่อ updater เป็น Device Owner** เท่านั้น
(`UpdateActivity.isDeviceOwner()` → `PackageInstaller` พร้อม `USER_ACTION_NOT_REQUIRED`)
ตั้งครั้งเดียวตอน provision เครื่อง:

```bash
adb shell dpm set-device-owner com.ttfts.vendingupdater/.DeviceOwnerReceiver
adb shell dumpsys device_policy        # ตรวจสอบสถานะ
```

ถ้าเครื่องมี user/บัญชีอยู่แล้ว คำสั่งนี้จะ fail — ต้องตั้งบนเครื่องที่ยังไม่ provisioned
(ดูขั้นตอนเต็มใน `vending-updater/README.md`)

---

## 5. Verify + Rollback

### 5.1 ยืนยันว่าเครื่องอัปเดตจริง

```bash
# versionCode/versionName ที่ติดตั้งอยู่บนเครื่อง
adb shell dumpsys package com.vendingmachine.app | grep -E 'versionCode|versionName'

# เทียบกับ versionCode ที่เพิ่ง bump ใน build.gradle.kts (เช่น 50)
```

ดู log การติดตั้งฝั่ง updater:

```bash
adb logcat -s UpdateActivity AutoLaunchReceiver InstallResultReceiver DeviceOwnerReceiver
```

ดู log การ check-update ฝั่ง kiosk (Timber tag `[AppUpdater]`):

```bash
adb logcat | grep AppUpdater
```

### 5.2 ถ้า install fail

ลำดับ fallback ใน `UpdateActivity.installApk()`:

1. **Device Owner** → `PackageInstaller` silent session (ทางหลัก)
2. ถ้าไม่ใช่ Device Owner → ลอง root: `su -c pm install -r <update.apk>`
   (สำเร็จเมื่อ exit code 0 + output มีคำว่า `Success`)
3. ถ้าทั้งคู่ fail → log error แล้ว `finish()` (เครื่องยังรันเวอร์ชันเดิม ไม่พัง)

อาการที่พบบ่อย (จาก README):

| Log message | สาเหตุ | แก้ |
|---|---|---|
| `No APK URL provided` | deep link ไม่มี `apkUrl` | ตรวจ URI ที่ยิง |
| `Install requires user action` | updater ยังไม่ใช่ Device Owner | `dpm set-device-owner` |
| `Root silent install failed` | เครื่องไม่ root / policy ห้าม | ใช้ Device Owner path |
| download fail | kiosk เข้า `apkUrl` ไม่ได้ | เช็ค baseUrl / network ของ site |

> หมายเหตุ: signature ต้องตรงกับที่ติดตั้งอยู่เดิม (ดูข้อ 2.2) ไม่งั้น `pm install -r` / PackageInstaller
> จะ reject ด้วย signature mismatch

### 5.3 Rollback

ไม่มีปุ่ม "downgrade" ตรง ๆ (เพราะ kiosk เทียบ `versionCode` สูงกว่าเท่านั้น) วิธี rollback:

1. ใน Web admin → App Versions → **toggle active** ปิดเวอร์ชันที่มีปัญหา (`PATCH /api/v1/portal/app-versions/:id/toggle-active`)
2. upload APK เวอร์ชัน "ใหม่" ที่มี **`versionCode` สูงกว่าเดิม** แต่เนื้อในเป็น code เก่าที่เสถียร
   (เพราะ kiosk จะไม่ติดตั้งเวอร์ชันที่ `versionCode` ต่ำกว่าที่รันอยู่)
3. ⚠️ verify: บนเครื่องที่ติดตั้งเวอร์ชันเสียไปแล้ว อาจต้อง manual push ด้วย adb (ข้อ 4.2) เพราะการ
   toggle ปิด active แค่หยุดเครื่องอื่นไม่ให้ดึงเวอร์ชันเสียเพิ่ม ไม่ได้สั่งเครื่องเก่าให้ถอย

### 5.4 Guard notes (ต้องเช็คทุก release)

- **`SerialService.USE_MOCK`** — release-safe โดย design: setter clamp เป็น `false` เสมอเมื่อไม่ใช่ debug
  (`_useMock = if (BuildConfig.DEBUG) value else false`) → release build จะ lock ไป VMC จริงเสมอ
  ห้ามแก้ logic นี้ให้เป็น plain `var` — เคยต้องตามปิด mock ก่อน prod (commit `443923d` + `1299433`)
  และเพิ่ม guard กัน mock เปิดเงียบ ๆ ใน release ภายหลัง (commit `427e844`)
- **`AppUpdater.checkForUpdate()`** ข้าม update check ทั้งหมดถ้า `BuildConfig.DEBUG` → OTA ทำงานเฉพาะ release build
- ออก release ต้องใช้ `assembleRelease` (signed + `BuildConfig.DEBUG=false`) เท่านั้น — อย่าเอา debug APK
  ขึ้นเครื่องจริง (mock เปิด + ไม่ check update)
- `isMinifyEnabled = false` ใน release build (ไม่ได้เปิด R8 shrink) — เป็นค่าปัจจุบันใน `build.gradle.kts`

---

## เอกสารที่เกี่ยวข้อง

- แผนอบรม internal — เอกสารนี้หนุน Session 4 (ดูแผนอบรม internal)
- [03-new-site-onboarding.md](03-new-site-onboarding.md) — onboard site/แบรนด์ใหม่ + provision เครื่อง (Device Owner)
- [01-system-architecture.md](01-system-architecture.md) · [02-debugging-runbook.md](02-debugging-runbook.md)
- `vending-updater/README.md` — รายละเอียด updater + Device Owner + troubleshooting (source of truth ฝั่งรับ OTA)
- `vending-machine-kotlin/KIOSK_SETUP.md` — install/signing บนเครื่อง kiosk
