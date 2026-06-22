# 01 — System Architecture & Integration Contract

> เอกสารอ้างอิงสำหรับนักพัฒนา (developer reference) ของระบบ vending-machine แบบ multi-repo
> เอกสารนี้รองรับ Session 3 ของแผนอบรม internal (Architecture + Stock + Debug)
>
> ทุกข้อมูลในเอกสารนี้ถูก verify จาก source จริง (กรณีที่ยังยืนยันไม่ได้จะใส่ `⚠️ verify` กำกับไว้)
> สำหรับรายละเอียด **Stock model** แบบเต็ม จะแยกเป็นเอกสารต่างหากในอนาคต — ที่นี่ให้แค่ภาพรวมพอเข้าใจ flow
> 📊 **Diagram แบบ render ได้ (Mermaid)** ดูที่ [../architecture-diagrams.md](../architecture-diagrams.md) — system / routing / deploy / OTA / stock / offline bring-up

---

## 1. ภาพรวมระบบ (Component Diagram)

ระบบประกอบด้วย 4 ส่วนหลักที่สื่อสารกันผ่าน REST + Socket.IO และมี backing store เป็น SQL Server + Redis

```
                          ┌──────────────────────────────┐
                          │   vending-machine-web         │
                          │   (React 18 + Vite SPA)       │
                          │   Admin / Operator portal     │
                          └───────────────┬──────────────┘
                                          │  REST (JWT auth)
                                          │  /api/v1/portal/*
                                          ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                       vending-machine-api                              │
   │                    (NestJS 11 + Fastify, ESM)                          │
   │                                                                        │
   │   /api/v1/portal/*   ← JWT + Permission guard  (Web)                   │
   │   /api/v1/machine/*  ← ApiKeyGuard (x-api-key)  (Kiosk REST)           │
   │   /ws/machine        ← Socket.IO 4.x  (Kiosk realtime, api-key)        │
   └───────┬───────────────────────────┬───────────────────────┬───────────┘
           │                           │                       │
   ┌───────▼────────┐         ┌────────▼───────┐       ┌───────▼────────┐
   │ SQL Server 2017│         │  Redis (cache) │       │  Mailpit (dev) │
   │ TypeORM 0.3.x  │         │  ioredis       │       │  SMTP 1025/8025│
   │ snake_case     │         │                │       │                │
   └────────────────┘         └────────────────┘       └────────────────┘
           ▲
           │  REST (/api/v1/machine/*) + Socket.IO (/ws/machine)
           │  Headers: x-api-key, x-machine-user-id
   ┌───────┴──────────────────────────┐
   │   vending-machine-kotlin          │
   │   (Android kiosk, Jetpack Compose)│
   │   Room (local SQLite) + Serial HW │
   └───────────────────────────────────┘
```

หมายเหตุ:
- **Mailpit** เป็น dev SMTP เท่านั้น (`docker compose` map `1025` = SMTP, `8025` = web UI) — production ใช้ SMTP จริง (`SMTP_URL`)
- Kiosk มี **Room (SQLite local)** เป็น offline store ของตัวเอง แล้ว sync ขึ้น API — ไม่ได้ต่อ SQL Server ตรง
- Web ไม่ได้คุยกับ kiosk โดยตรง ทุกอย่างผ่าน API เป็นตัวกลาง (รวมถึง remote command → kiosk ที่ส่งผ่าน Socket.IO room)

### Tech Stack (at a glance)

สรุปสแต็กหลักของ 3 แอป — รายละเอียด folder + pattern อยู่ §7, source of truth คือ `AGENTS.md` ของแต่ละ repo

| Layer | Repo | Core | Architecture |
|---|---|---|---|
| **API** (backend) | `vending-machine-api` | NestJS 11 + Fastify · TypeScript (ESM) | Clean / Hexagonal + **DDD** (feature module 3 ชั้น: application / domain / infrastructure) |
| **Web** (admin) | `vending-machine-web` | React 18 + **Vite** 7 SPA · TypeScript strict · pnpm | feature-based SPA |
| **Kiosk** (Android) | `vending-machine-kotlin` | Kotlin 2.1 · Jetpack Compose | Clean Arch: UI → ViewModel → Repository |

**Web — React ecosystem** (ตาม diagram):

| ด้าน | ไลบรารี |
|---|---|
| API / server state | TanStack **React Query** (+ axios `apiClient`, auto refresh-token) |
| Routing | TanStack Router (file-based, type-safe) |
| Form + validate | **React Hook Form** + **Zod** (map error → field ผ่าน `lib/form-errors.ts`) |
| UI component | **shadcn/ui** (Radix primitives) |
| Style | **Tailwind** CSS v4 (Biome auto-sort classes) |
| Local state / i18n | Jotai (UI) · React Context (auth) · i18next (en/th/ja) |

> API stack เพิ่มเติม: TypeORM + MSSQL 2017 · Redis · Socket.IO · JWT + API Key · Pino · i18n.
> Kiosk stack เพิ่มเติม: Hilt · Room (SQLite) · Retrofit + OkHttp · Socket.IO · USB Serial (JNI).

---

## 2. ช่องทางสื่อสาร (Communication Channels)

### 2.1 REST

Global prefix = `/api`, URI versioning = `v1`, แยก 2 base ผ่าน `RouterModule` (`path: 'portal'` / `path: 'machine'`)

**Portal API — `/api/v1/portal/*`** (JWT + permission-based)
กลุ่ม route หลัก (จาก `@Controller` base path ใน `src/api/portal/*`):

`auth`, `users`, `roles`, `permissions`, `permission-groups`, `machines`, `products`, `stocks`,
`issue-slips`, `refill-slips`, `kanbans`, `kanban-levels`, `warehouses`, `departments`, `sections`,
`reasons`, `tags`, `units`, `advertisements`, `themes`, `languages`, `translations`, `manuals`,
`files`, `app-versions`, `audit-logs`, `dashboard`, `alerts`, `alert-channels`,
`alert-notification-histories`, `smtp-settings`, `line-oa-settings`, `slot-lock-histories`

> การสร้าง/register machine และ regen api-key อยู่ใต้ `POST /api/v1/portal/machines` และ
> `POST /api/v1/portal/machines/:id/regenerate-api-key` ทั้งคู่ต้องตรงกับ env `MACHINE_CREATE_PASSWORD`

**Machine API — `/api/v1/machine/*`** (`@Public()` + `@UseGuards(ApiKeyGuard)`)
route จริงจาก `src/api/machine/machine.controller.ts` จัดกลุ่มตามหน้าที่:

| หมวด | Method + Path | หน้าที่ |
|---|---|---|
| Heartbeat | `POST /heartbeat` | ส่ง ping → คืน `{ lastHeartbeatAt }` (อัปเดต `machines.last_heartbeat_at`) |
| Sync (อ่าน) | `GET /theme` `GET /languages` `GET /translations` `GET /advertisements` `GET /reasons` | ดึง config / สื่อ / theme ของเครื่อง |
| Sync (อ่าน) | `GET /products` `GET /slots` `GET /kanbans` `GET /users` `GET /stock-movements` | ดึง catalog / slot / kanban / user / movement |
| Issue slip (อ่าน) | `GET /issue-slips` | ดึง issue slip (status `pending` / `partial_picked`) |
| Refill slip (อ่าน) | `GET /refill-slips` | ดึง refill slip (status `submitted`) |
| Pickup / dispense | `POST /pickup` | รายงานการหยิบของ (`MachinePickupIssueSlipDto`) |
| Pickup / dispense | `POST /issue-slips/:id/items/:itemId/pickup` | pickup item เฉพาะรายการ |
| Pickup / dispense | `POST /dispense-success` `POST /dispense-failed` | รายงานผลการ dispense ต่อครั้ง |
| Stock / lock | `POST /stock-movements` `POST /slot-locks` | บันทึก movement / slot-lock history |
| Issue slip (เขียน) | `PATCH /issue-slips/:id/cancel` `PATCH /issue-slips/:id/complete` `DELETE /issue-slips/:id/items/:itemId` | ยกเลิก / ปิด / ลบรายการ |
| Refill slip (เขียน) | `PATCH /refill-slips/:id/cancel` `PATCH /refill-slips/:id/complete` | ยกเลิก / ปิด refill slip |

App-update (แยก controller, ไม่ต้องใช้ api-key): `GET /api/v1/machine/app-update/check?appName=&versionCode=`

> หมายเหตุ: machine **ไม่มี** REST sign-in endpoint แยก — การ authenticate machine ใช้ api-key (REST header + Socket.IO connect). ส่วน "การ login ของ operator" ที่ kiosk จะอ้างถึง user ผ่าน header `x-machine-user-id` (ดูข้อ 3)

### 2.2 Socket.IO

- Namespace / path: **`/ws/machine`** (Socket.IO 4.8.x), CORS `{ origin: true, credentials: true }`
- การ join: หลัง authenticate สำเร็จ client จะ join room **`machine:${machineId}`**
- api-key ส่งได้ทาง header / auth object / query (kiosk ส่งทาง `extraHeaders` → `x-api-key`)
- ชื่อ event ทั้งหมดนิยามใน enum `MachineSocketEvent`
  (`src/shared/domain/enums/machine-socket-event.enum.ts`) — convention คือ `<domain>:<action>`

**Server → Client** (API ส่งหา kiosk; ส่วนมาก emit เข้า room ของเครื่อง):

| Event (string) | ความหมาย |
|---|---|
| `machine:info` | ส่งข้อมูล machine หลัง connect |
| `machine:lock` / `machine:unlock` | ล็อก / ปลดล็อกเครื่อง |
| `machine:products:submitted` | product slots อัปเดต |
| `machine:advertisements:submitted` | advertisements อัปเดต |
| `machine:themes:submitted` | theme อัปเดต |
| `machine:languages:submitted` | languages อัปเดต |
| `reason:submitted` | reason อัปเดต |
| `user:submitted` | user เพิ่ม/แก้ไข/เปลี่ยน pin |
| `refill-slip:submitted` / `refill-slip:executed` | refill slip submit แล้ว / execute แล้ว (stock อัปเดต) |
| `issue-slip:submitted` / `issue-slip:deleted` | issue slip submit / ถูกลบ |
| `kanban:submitted` | kanban submit แล้ว |
| `machine:slot-unlock:submitted` | สั่ง unlock slot |
| `machine:sync:request` | สั่ง sync ข้อมูลใหม่ทั้งหมด |
| `machine:database:reset:request` | สั่ง reset (ลบ) local DB แล้ว sync ใหม่ |
| `machine:force-replay-pickup:request` | สั่ง re-queue pending pickup op (กรณี orphan) |
| `machine:tailscale:restart` | สั่ง restart Tailscale |
| `database:query:request` | สั่ง kiosk query local DB (monitor) |
| `error` | แจ้ง error (เช่น auth fail) |

**Client → Server** (kiosk ส่งหา API):

| Event (string) | ความหมาย |
|---|---|
| `machine:info:request` | ขอข้อมูล machine |
| `kanban:sync:ack` | ยืนยันว่า sync kanban payload ล่าสุดเสร็จแล้ว |
| `database:query:response` | ส่งผลลัพธ์ database query กลับ (คู่กับ `database:query:request`) |
| `client:going-down` | สัญญาณว่าตู้กำลังจะปิด/หลุดเอง (payload `{ reason }` เช่น `'app-update'`) — server เก็บ reason ไว้ใช้เป็น `event_source` ตอน disconnect |

---

## 3. Auth Model

### 3.1 สองโลกของ auth
- **Portal (Web):** JWT (access + refresh) + Passport → `JwtAuthGuard` → `PermissionsGuard` → `RolesGuard`
- **Machine (Kiosk):** **api-key** (bcrypt hash) ผ่าน `ApiKeyGuard` ทั้ง REST และ Socket.IO

### 3.2 Machine: api-key + `x-machine-user-id`
ทุก request จาก kiosk แนบ 2 header (จาก `ApiKeyInterceptor.kt` ฝั่ง Kotlin):
- `x-api-key` — ยืนยันว่าเป็น **เครื่องไหน** (`ApiKeyGuard` เทียบ hash กับ `machines.api_key_hash` ทุก entry)
- `x-machine-user-id` — ยืนยันว่า **operator คนไหน** กำลังทำรายการ

flow ใน `ApiKeyGuard` (`src/modules/machine/presentation/guards/api-key.guard.ts`):

```text
1. อ่าน x-api-key → เทียบ hash → ได้ machine → แนบ request.machine
2. ถ้ามี x-machine-user-id → contextService.setCurrentUser({
       sub: machineUserId,
       sid: `machine:<machineId>`,
       email: '', permissions: [],
   })
3. use case อ่านผู้ทำรายการผ่าน contextService.getCurrentUserId()
```

### 3.3 ทำไม "ไม่ส่ง header = UUID สุ่มต่อ call"
`getCurrentUserId()` สร้าง `UserId.create(user?.sub)`. ถ้าไม่ได้ `setCurrentUser` (เพราะไม่มี `x-machine-user-id`)
ค่า `sub` เป็น `undefined` → `UniqueEntityID` constructor **gen UUIDv7 สุ่มใหม่ทุกครั้งที่เรียก** →
แต่ละ call มองเป็นคนละ user → ownership check เช่น `PickupIssueSlipItemUseCase` จะ throw
`IssueSlipUnauthorizedPickupException`.
สรุป: **ถ้าลืมส่ง `x-machine-user-id` การ pickup จะ unauthorized แบบสุ่ม** — เป็น pitfall ที่ต้องระวัง

> endpoint ที่ไม่ต้องผูก user (heartbeat, advertisements ฯลฯ) ส่ง header นี้หรือไม่ก็ได้

---

## 4. Data Flow Examples

### 4.1 Machine connect / sign-in (api-key handshake)
```
Kiosk ─ Socket.IO connect /ws/machine (x-api-key) ─▶ API
API   ─ ApiKeyGuard เทียบ hash machines.api_key_hash
API   ─ client.join("machine:<id>")  +  emit "machine:info"  ─▶ Kiosk
Kiosk ─ POST /api/v1/machine/heartbeat (เป็นระยะ) ─▶ API → UPDATE machines.last_heartbeat_at
```
หมายเหตุ: Monitor page ดูจาก live socket map ส่วน Manage page ดูจาก `last_heartbeat_at` (90s) — สอง path นี้แยกกัน

### 4.2 Product / stock sync (server → kiosk)
```
Web ─ แก้ product/slot (REST /api/v1/portal/products|stocks) ─▶ API → SQL Server
API ─ emit "machine:products:submitted" เข้า room "machine:<id>" ─▶ Kiosk
Kiosk ─ รับ payload → เขียนลง Room (local SQLite) → UI (Compose) อัปเดตผ่าน Flow
```
หรือ kiosk pull เองตอน boot/refresh ผ่าน `GET /api/v1/machine/products` + `GET /slots`

### 4.3 Dispense / pickup report (kiosk → server)
```
Kiosk ─ จ่ายของผ่าน Serial HW (USB/JNI) → ผลสำเร็จต่อชิ้น
Kiosk ─ POST /api/v1/machine/dispense-success | dispense-failed ─▶ API   (รายงานผล dispense)
Kiosk ─ POST /api/v1/machine/pickup (x-machine-user-id) ─▶ API
        └ ApiKeyGuard.setCurrentUser → PickupIssueSlipUseCase ผูก picked_by_id
API ─ บันทึก issue_slip_pickups + stock_movements → SQL Server
```
ถ้า offline: kiosk queue ไว้ใน `PendingOperationEntity` แล้ว replay ตอน sync รอบถัดไป

---

## 5. Stock Model (ภาพรวมย่อ)

> รายละเอียดเต็ม (merged slots, tray badge, refill math) จะอยู่ในเอกสาร stock แยกต่างหากในอนาคต

- **สอง “ที่อยู่” ของ stock:** ตู้ (machine/slot) กับ warehouse — เอกสารนี้โฟกัสฝั่งตู้
- **`slots` / `slot_products`:** stock จริงในตู้ผูกกับ slot (มี `current_stock`, `merge_role`, `merged_with_slot_ids`)
  ฝั่ง kiosk stock จริงของสินค้า = `SUM(slot_products.stock)` group by product (ไม่ใช่ remaining ของ issue-slip)
- **Sync snapshot + `maxOf`:** `syncIssueSlips()` ทำ delete-all + insert จาก API ซึ่งจะทับ `picked_quantity` local
  ดังนั้นต้อง snapshot ค่า local ก่อน แล้วใช้ `maxOf(local, remote)` ตอน re-insert
- **`picked_quantity` conflict:** เป็น pitfall หลักของ sync — local อาจ pick ไปแล้วแต่ remote ยังเป็นค่าเก่า
  อย่าให้ค่า remote ที่ต่ำกว่ามาทับค่า local ที่สูงกว่า

---

## 6. Repo Map

| Repo | บทบาท (1 บรรทัด) |
|---|---|
| `vending-machine-api` | Backend NestJS + Socket.IO gateway — REST `/portal` (JWT) + `/machine` (api-key), เจ้าของ SQL Server schema |
| `vending-machine-web` | Admin/operator portal (React 18 + Vite SPA) คุยกับ `/api/v1/portal/*` |
| `vending-machine-kotlin` | Android kiosk app (Jetpack Compose + Room + Serial HW) คุยกับ `/api/v1/machine/*` + `/ws/machine` |
| `vending-machine-gitops` | Template-based deploy (blue-green, Windows Server) — render env, secrets อยู่บน server |
| `vending-updater` | OTA / silent installer สำหรับ push app ใหม่ลงตู้ |
| `vending-machine-elk` | Observability stack (Elasticsearch + Logstash + Kibana) — เก็บ server logs |
| `vending-machine-status` | Status / uptime page ของระบบ |

> หมายเหตุ: kiosk dispense logs **ไม่ขึ้น Kibana** — ต้องดูที่ `adb logcat | grep VMC_SERIAL` (Kibana เก็บเฉพาะ server logs)

---

## 7. โครงสร้างโค้ดภายใน — BE / FE / Kiosk

> สรุประดับ folder + แพทเทิร์นหลัก ของ source code 3 repo รายละเอียดเชิงลึก (ทุก pattern, naming, command) อยู่ที่ `AGENTS.md` ของแต่ละ repo — ถือเป็น source of truth, ไฟล์นี้สรุปให้เห็นภาพ

### 7.1 Backend — `vending-machine-api`

**Stack:** NestJS 11 + Fastify · TypeScript (ESM) · TypeORM + MSSQL 2017 (prod; ver per customer) · Redis · Socket.IO · JWT + API Key · Pino · i18n (en/th/ja)

**Clean / Hexagonal architecture** — แยกตาม feature module, แต่ละ module มี 3 ชั้น:

```
src/
├── api/                 # ชั้น HTTP/WS (controllers, gateway)
│   ├── machine/         # Machine device API — api-key auth (controller + gateway + app-update)
│   └── portal/          # Admin/Web API — JWT auth (33 controllers)
├── modules/             # 23 feature modules — โค้ดธุรกิจอยู่ที่นี่
│   └── {feature}/
│       ├── application/     # use-cases (1 class / 1 action), dto, ports
│       ├── domain/          # entities, repository interfaces, value-objects
│       ├── infrastructure/  # TypeORM orm-entities, repo impl, passport strategies
│       └── *.module.ts
├── shared/              # config, database (126 migrations + seeds), filters, guards,
│                        #   decorators, i18n, logger, cache(Redis), context(CLS)
└── test/                # 21 E2E suites
```

- **23 feature modules:** auth, user, role, permission, permission-group, machine, product, stock, issue-slip, refill-slip, kanban, warehouse, section, department, alert, advertisement, app-version, language, theme, reason, file, manual, audit-log
- **Use-case pattern:** 1 action = 1 `{Action}{Feature}UseCase` class
- **Repository pattern:** interface ใน `domain/repositories/` → impl ใน `infrastructure/repositories/` inject ผ่าน token
- **Guard chain (global):** `JwtAuthGuard` (`@Public()` ข้าม) → `PermissionsGuard` → `RolesGuard` → `ApiKeyGuard` (machine API)
- **Routes:** `/api/v1/portal/*` (JWT) · `/api/v1/machine/*` (api-key) · `/ws/machine` (Socket.IO)

→ ลึกกว่านี้: `vending-machine-api/AGENTS.md`

### 7.2 Frontend (Admin Web) — `vending-machine-web`

**Stack:** React 18 + Vite 7 SPA · TypeScript strict · TanStack Router (file-based) + React Query · shadcn/ui (Radix + Tailwind v4) · Jotai · React Hook Form + Zod · i18next · pnpm

```
src/
├── routes/              # file-based routes (TanStack Router)
│   ├── (auth)/          # login/forgot/reset (URL group, ไม่มี path segment)
│   ├── (app)/           # protected routes
│   └── routeTree.gen.ts # AUTO-GEN — ห้ามแก้
├── components/
│   ├── ui/              # shadcn/ui primitives
│   ├── common/          # reusable (Page, TextField, Combobox, ConfirmDialog, …)
│   └── features/        # feature-specific (Product, Machine, Kanban, Refill, …)
├── services/
│   ├── hooks/           # API hooks ต่อ domain (use{Resource} query, use{Action}{Resource} mutation)
│   ├── models/          # TS interfaces ของ API entity
│   └── types.ts         # PaginatedResponse<T>, DataResponse<T>, …
├── lib/                 # utils, permissions, form-errors, contexts/ (auth)
├── config/  i18n/  locales/{en,th,ja}/  icons/  integrations/ (Sentry, RQ provider)
```

- **API layer:** axios instance เดียวใน `apiClient.ts` — ใส่ auth header + refresh token อัตโนมัติเมื่อ 401; endpoint default ใต้ `/portal/{resource}`
- **State แบ่ง 3:** Jotai (UI local) · React Context (auth, token ใน localStorage/sessionStorage) · React Query (server state)
- **Forms:** React Hook Form + Zod; map API validation error → field ผ่าน `lib/form-errors.ts`

→ ลึกกว่านี้: `vending-machine-web/AGENTS.md` + `CONTEXT.md`

### 7.3 Kiosk (Android) — `vending-machine-kotlin`

**Stack:** Kotlin 2.1 · Jetpack Compose (Material 3) · Hilt · Room (SQLite) · Retrofit + OkHttp · Socket.IO · USB Serial (JNI)

**Clean Architecture:** UI (Compose) → ViewModel → Repository → (Room / Retrofit)

```
app/src/main/java/com/vendingmachine/app/
├── di/          # Hilt modules (Network, Database)
├── ui/          # navigation, theme, screens/ (home,cart,delivery,pickup,refill,…), components/
├── data/        # local/ (Room: 22 entities, 22 DAOs) · remote/ (Retrofit DTOs, interceptors)
│                #   preferences/ (DataStore) · repository/
├── service/     # socket/ (MachineSocketService) · serial/ (SerialService — HW)
├── util/        # Translation, Kiosk, Scan, Auth, EventBus, Session
└── receiver/    # BootReceiver, KioskDeviceAdminReceiver
```

- **Screen pattern:** `FeatureScreen` (stateful, ViewModel) → `FeatureContent` (stateless, preview)
- **Auth:** ส่ง header `x-machine-user-id` ทุก request (ดู §3.2) ผ่าน `ApiKeyInterceptor`
- **Offline:** API fail → คิวลง `PendingOperationEntity`, replay ตอน sync รอบถัดไป
- หน้าจอ portrait 1080×1920 — layout เป็น Column เสมอ (ห้าม Row ระดับ page)

→ ลึกกว่านี้: `vending-machine-kotlin/AGENTS.md`

---

## อ้างอิง source (verify trail)

- Socket events: `vending-machine-api/src/shared/domain/enums/machine-socket-event.enum.ts`
- Machine REST: `vending-machine-api/src/api/machine/machine.controller.ts`, `app-update.controller.ts`
- Auth guard: `vending-machine-api/src/modules/machine/presentation/guards/api-key.guard.ts`
- Portal controllers: `vending-machine-api/src/api/portal/*.controller.ts`
- Router paths: `vending-machine-api/src/app.module.ts` (`path: 'portal' | 'machine'`)
- Kiosk headers: `vending-machine-kotlin/app/src/main/java/com/vendingmachine/app/data/remote/ApiKeyInterceptor.kt`
- Schema: `vending-machine-api/docs/er-diagram.mmd`, `docs/schema.dbml`
