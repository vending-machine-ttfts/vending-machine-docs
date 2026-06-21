# Architecture & Flow Diagrams (Mermaid)

> Diagram ภาพรวมระบบ vending-machine สำหรับ **Session 3 (architecture + stock)** และ **Session 4 (deploy/OTA)**
> ทุก endpoint / port / behavior verify จาก source จริงแล้ว (ดู [`dev-guidelines/01-system-architecture.md`](dev-guidelines/01-system-architecture.md) สำหรับรายละเอียด)
> Mermaid render ได้ใน GitHub + VSCode (ติดตั้ง *Markdown Preview Mermaid Support* ถ้ายังไม่ขึ้น)

สารบัญ:
1. [System Architecture](#1-system-architecture) — ส่วนประกอบ + ช่องทางคุยกัน
2. [API Routing & Auth](#2-api-routing--auth) — portal vs machine vs socket
3. [GitOps Deploy Pipeline](#3-gitops-deploy-pipeline) — code → kiosk/web ขึ้น production
4. [Blue-Green Deploy](#4-blue-green-deploy-detail) — `deploy.ps1` ทำงานยังไง
5. [Android OTA](#5-android-ota-flow) — ปล่อย APK เวอร์ชันใหม่ให้ตู้
6. [Stock & Pickup Dispense](#6-stock--pickup-dispense-flow) — `stocks.quantity` = source of truth
7. [Offline Bring-up (ลูกค้า offline)](#7-offline-bring-up-ลูกค้า-offline-sequence) — flow วันส่งมอบ (map กับ runbook)

---

## 1. System Architecture

4 ส่วนหลัก คุยกันผ่าน **REST + Socket.IO**, backing store = **SQL Server (native) + Redis**.

```mermaid
flowchart TB
    subgraph client[Clients]
        WEB["vending-machine-web<br/>React 18 + Vite SPA<br/>Admin / Operator portal"]
        KIOSK["vending-machine-kotlin<br/>Android kiosk (Compose)<br/>Room SQLite + Serial HW"]
    end

    API["vending-machine-api<br/>NestJS 11 + Fastify<br/>prefix /api · version v1"]

    subgraph store[Backing store]
        DB[("SQL Server 2017<br/>native on Windows<br/>TypeORM")]
        REDIS[("Redis<br/>NSSM service")]
        MAIL[("Mailpit / SMTP<br/>NSSM service")]
    end

    HW["Serial hardware<br/>(USB / SerialPort JNI)"]

    WEB -->|"REST /api/v1/portal/* · JWT"| API
    KIOSK -->|"REST /api/v1/machine/* · x-api-key + x-machine-user-id"| API
    KIOSK <-->|"Socket.IO /ws/machine · realtime"| API
    KIOSK <-->|"dispense / status"| HW
    API --> DB
    API --> REDIS
    API --> MAIL
```

- Web **ไม่คุยกับ kiosk ตรง** — ทุกอย่างผ่าน API (remote command → kiosk ส่งผ่าน Socket.IO room)
- Kiosk มี **Room (SQLite local)** เป็น offline store → sync ขึ้น API (ไม่ต่อ SQL Server ตรง)
- prod: SQL Server = **native** (<IT contact> ลงให้), Redis + Mailpit = **NSSM Windows service** (ไม่ใช่ container)

---

## 2. API Routing & Auth

API global prefix `/api`, URI version `v1`, แยก 2 base ผ่าน `RouterModule` + 1 socket namespace.

```mermaid
flowchart LR
    WEB["Web (admin)"] -->|JWT + Permission guard| PORTAL["/api/v1/portal/*"]
    KIOSK["Kiosk REST"] -->|"ApiKeyGuard (x-api-key)"| MACHINE["/api/v1/machine/*"]
    KIOSK2["Kiosk realtime"] -->|"api-key"| WS["/ws/machine (Socket.IO)"]

    subgraph nest[NestJS app]
        PORTAL --> PMOD[PortalApiModule]
        MACHINE --> MMOD[MachineApiModule]
        WS --> GW[MachineGateway]
    end

    MMOD -. "x-machine-user-id → contextService.setCurrentUser()" .-> CTX["acting user<br/>(ไม่ส่ง = random UUID/call)"]
```

- `x-machine-user-id` สำคัญ: ApiKeyGuard อ่าน header → set acting user → use case (เช่น pickup) รู้ว่าใครทำ. ถ้าไม่ส่ง → `getCurrentUserId()` คืน UUID สุ่มทุก call → `IssueSlipUnauthorizedPickupException`
- ตัวอย่าง endpoint (verified): `POST /api/v1/portal/app-versions`, `PUT /api/v1/portal/themes/machines/{machineId}`

---

## 3. GitOps Deploy Pipeline

จาก commit → image → ขึ้น production (web/api). **ตู้ Android ไปทาง OTA แยก** (ดู §5).

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant GH as GitHub
    participant CI as GitHub Actions<br/>(windows-container-build.yml)
    participant HUB as Docker Hub<br/>(<docker-user>/*)
    participant SRV as Windows Server<br/>(deploy.ps1 / NSSM)
    participant IIS as IIS + ARR

    Dev->>GH: push (api / web)
    GH->>CI: trigger build
    CI->>CI: docker build -f Dockerfile.windows<br/>(nanoserver:ltsc2025)
    CI->>HUB: push image :tag
    Dev->>GH: bump image tag ใน gitops compose
    Note over SRV: deploy.ps1 loop ทุก 60s
    SRV->>GH: git pull (gitops)
    SRV->>SRV: render compose+env → MD5 hash
    alt hash เปลี่ยน
        SRV->>HUB: docker compose pull
        SRV->>SRV: blue-green up -d (ดู §4)
        SRV->>IIS: สลับ ARR ไป slot ใหม่
    else hash เดิม
        SRV->>SRV: "No change — skip"
    end
```

- **prod = Windows container** (CI build จาก `Dockerfile.windows`). gitops network = `nat` driver
- deploy ทำงานเมื่อ **compose-hash เปลี่ยน** (มาจาก git pull). offline (ลูกค้า <customer> หลังส่งมอบ) → git/pull fail = idle ไม่กระทบ service ที่รันอยู่

---

## 4. Blue-Green Deploy (detail)

`deploy.ps1` ดึง image แล้วขึ้น slot สำรอง, health ผ่านค่อยสลับ — ไม่มี downtime.

```mermaid
flowchart TB
    START([deploy tick]) --> PULL["docker compose pull"]
    PULL --> UP["up -d slot สำรอง<br/>blue 8080/8081 ⇄ green 8082/8083"]
    UP --> HEALTH{"health ok?<br/>GET /api/v1/health"}
    HEALTH -->|yes| SWAP["IIS ARR → slot ใหม่<br/>(80/443 → active)"]
    HEALTH -->|"no (timeout 180s)"| KEEP["คง slot เดิมไว้<br/>(deploy ล้มเหลว ไม่ตัด traffic)"]
    SWAP --> DONE([active = slot ใหม่])
    KEEP --> DONE2([active = slot เดิม])
```

- pull fail (offline) ถูก tolerate — `up -d` รันจาก image ที่ load ไว้แล้ว (`$ok` อ่าน exit code ของ `up -d`)
- พังหน้างาน → ดู recovery card ในชุดเอกสาร internal (live-demo-recovery-card)

---

## 5. Android OTA Flow

ปล่อยแอปเวอร์ชันใหม่ → ตู้ดึงเอง. **all-or-nothing ต่อ `appName`** (ไม่มี per-machine target).

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant WEB as Web portal
    participant API as API (/portal/app-versions)
    participant K as Kiosk (AppUpdater)

    Dev->>Dev: build + sign release APK<br/>(versionCode ต้องเพิ่มเสมอ)
    Dev->>WEB: upload APK
    WEB->>API: POST /api/v1/portal/app-versions<br/>(multipart file)
    API->>API: save /uploads/apk/<file><br/>set isActive
    loop ตู้ poll
        K->>API: GET list (filter appName, isActive, max versionCode)
        alt remote.versionCode > installed
            K->>API: download APK
            K->>K: silent install (device owner)
        else เท่าเดิม
            K->>K: ไม่ทำอะไร
        end
    end
```

- rollback ติดกฎ `versionCode` ต้องเพิ่ม → ลดเวอร์ชันต้อง toggle active + เครื่องที่อัปไปแล้วอาจต้อง `adb` ลดเอง
- DEBUG build ข้าม OTA

---

## 6. Stock & Pickup Dispense Flow

**Source of truth = `stocks.quantity`** (table `stocks`). `slots.current_stock` = denormalized mirror (runtime ไม่เขียน). Issue Slip สร้างจาก **web เท่านั้น** — ตู้แค่ pickup/จ่าย.

```mermaid
flowchart TB
    ADMIN["Admin (web)<br/>สร้าง Issue Slip"] -->|"/portal"| APIDB[("stocks.quantity")]

    subgraph kioskflow[ตู้ Android — Pickup]
        OPEN["เปิด tab Pickup<br/>เลือก slip"] --> CONF["ยืนยัน<br/>(reason / tool-life)"]
        CONF --> DEL["route → Delivery screen<br/>SerialService dispense"]
        DEL --> REP["reportPickup()"]
    end

    APIDB --> OPEN
    REP -->|online| UC["pickup-issue-slip-item.use-case<br/>findBySlotId → check qty<br/>removeQuantity → save"]
    UC --> APIDB
    REP -->|offline| Q["queue PendingOperation<br/>(TYPE_PICKUP_ISSUE_SLIP)<br/>replay ตอน sync"]
    Q -.-> UC
```

- Tool Life บังคับ 1 ชิ้น/รายการ; reason บังคับเมื่อ slip กำหนด
- mock mode (`USE_MOCK=true`, dev เท่านั้น) ข้าม API call แต่ยัง `updateLocalPickedQuantity()`
- รายละเอียด: [`dev-guidelines/06-stock-domain-model.md`](dev-guidelines/06-stock-domain-model.md)

---

## 7. Offline Bring-up (ลูกค้า offline) (sequence)

flow วันส่งมอบหน้างาน — map กับ runbook offline bring-up (ลูกค้า offline) ในชุดเอกสาร internal. Server **มีเน็ตตอน setup**, offline หลังส่งมอบ.

```mermaid
sequenceDiagram
    autonumber
    participant T as Trainer
    participant PC as <customer> PC (online ตอน setup)
    participant DB as SQL Server (native, <IT contact> ลง)
    participant HUB as Docker Hub

    T->>PC: setup-server.ps1<br/>(Windows Containers + Docker + Git + NSSM + IIS)
    T->>DB: verify TCP 1433 + SQL auth + sa
    T->>PC: setup-site.ps1 -Brand <brand> -CreateDatabase -SqlServerInstance localhost
    T->>PC: init-secrets.ps1<br/>(DB_HOST=machine name, VITE_API_KEY, ...)
    T->>PC: nssm start <brand>-gitops-deploy
    PC->>HUB: docker compose pull (online OK)
    PC->>DB: API entrypoint → typeorm migration:run (auto)
    PC->>PC: up -d blue → health ok
    T->>PC: doctor.ps1 -Brand <brand> (FAIL=0)
    Note over PC: ── ส่งมอบ → ตัดเน็ต ──
    PC-->>HUB: git/compose pull fail ทุก tick<br/>(harmless — service รันต่อ)
```

- ไม่ต้อง `docker save`/`load` — pull ตอน online ได้เลย
- `DB_HOST` = **ชื่อเครื่อง / NAT-gateway IP** (host.docker.internal ไม่ resolve) → ดู [`tech-team-handbook.md`](tech-team-handbook.md) §1 (server/native DB wiring)
