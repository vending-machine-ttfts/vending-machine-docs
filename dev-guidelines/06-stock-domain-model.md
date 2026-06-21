# 06 — Stock Domain Model Reference

> เอกสารอ้างอิงสำหรับนักพัฒนา (developer reference) — โมเดล stock ของระบบ vending-machine
> รองรับ **Session 3** (Architecture + Stock + Debug, ดูแผนอบรม internal) และเป็น prerequisite ของการอบรม **Session 5 (อนาคต) — Return Stock / Tool Room** ที่จะจัดหลัง Phase 1 ส่งมอบ
>
> ทุกกฎในเอกสารนี้ verify จาก source จริง: `vending-machine-kotlin/AGENTS.md`, `vending-machine-api/docs/schema.dbml`, และ PRD Return Stock (Phase 1–3, ดู PRD ในชุดเอกสาร dev) — กรณีที่ยืนยันจาก source ไม่ได้จะกำกับ `⚠️ verify`
> ดูภาพรวมระบบ + ช่องทางคุยกันก่อนที่ [01-system-architecture.md](01-system-architecture.md)

---

## ก่อนเริ่ม: คำศัพท์ที่ชนกันบ่อย

ในระบบนี้มีคำว่า "stock" หลายระดับ และ **ตารางคนละตัวเก็บค่าคนละความหมาย** ต้องแยกให้ออกก่อนอ่านโค้ด:

| คำ / ฟิลด์ | อยู่ที่ไหน | ความหมายจริง |
|---|---|---|
| physical stock (ในตู้จริง) | ตู้ Kotlin: `slot_products.stock` (Room) ; API: `stocks.quantity` (source of truth) — `slots.current_stock` เป็น mirror | ของในช่อง (slot) ที่จับต้องได้จริง |
| stock รวมต่อสินค้า | คำนวณ (aggregate) = `SUM(stock)` group by `product_id` | ยอดรวมที่เอาไปโชว์คลังสินค้า / เทียบ min |
| `remainingQuantity` (API) | DTO ของใบเบิก (issue slip) | จำนวน **คงเหลือในใบเบิก** ไม่ใช่ stock จริง — **อย่าใช้แทน physical stock** |
| `minimum_stock` / `min_stock_threshold` | `products` + `machine_products` | เกณฑ์ขั้นต่ำ (ของน้อยกว่านี้ = badge เตือน) |

> หัวใจของ Phase ปัจจุบัน: **stock จริงอยู่ที่ระดับ slot เท่านั้น** ไม่มี "คลังสินค้า" เป็นก้อนแยก — หน้าคลังคือ aggregate view ของ slot ทั้งหมด

---

## 1. โมเดลปัจจุบัน — stock อยู่ที่ไหนจริง

### 1.1 โครงสร้าง location (จาก `schema.dbml`)

ลำดับชั้นของตู้จริง (ฝั่ง API):

```
machines → containers → trays → slots
                                  └─ slots.current_stock   (int, mirror ของ stocks.quantity)
                                  └─ slots.product_id      (ช่องนี้ผูกสินค้าอะไร)
                                  └─ slots.low_stock_threshold
                                  └─ slots.merge_role / merged_with_slot_ids (json)  ← slot merge
```

ตารางที่เก็บ stock จริงฝั่ง API:

- `stocks.quantity` — 1:1 กับ slot (`stocks.slot_id` unique) — **เป็น source of truth ของ stock ฝั่ง API**. pickup อ่าน/หักจากตารางนี้โดยตรง (`pickup-issue-slip-item.use-case.ts` → `stockRepository.findBySlotId()` เช็ค `stock.quantity >= quantity` → `stock.removeQuantity()` → save) ; refill/adjust/dashboard ก็ใช้ `stocks.quantity` เหมือนกัน (`@Entity('stocks')`)
- `slots.current_stock` — denormalized mirror บน slot ; **โค้ด runtime ไม่ได้เขียนค่านี้** (มีแต่ seed/migration set ตอนเตรียมข้อมูล) → ใช้อ่านเล่น ๆ ได้ แต่ของจริงให้ยึด `stocks.quantity`
- `stock_movements` — ledger การเคลื่อนของทุกครั้ง (`movement_type`, `quantity`, `previous_quantity`, `new_quantity`, `tool_life`, `created_by_id`) — เป็นแหล่ง audit/ประวัติ

ฝั่งตู้ (Kotlin / Room) เก็บ stock จริงไว้ที่ **`slot_products.stock`** (entity `SlotProductEntity`) ซึ่ง sync มาจาก API แล้วใช้เป็น offline store ของตู้เอง

### 1.2 ตู้ (per-slot) vs คลังสินค้า (aggregate view)

- **Per-slot (ของจริง):** หนึ่งสินค้าอาจกระจายอยู่หลาย slot → stock จริงคือผลรวมข้าม slot
- **คลังสินค้า (aggregate):** ไม่ใช่ตารางแยก — คือ query รวม `SUM(slot_products.stock) GROUP BY product_id`

กฎที่ verify จาก `AGENTS.md`:

> **Stock source:** `PickupDetailViewModel.stockMap` uses **actual** stock from `slotProductDao.getAllProductStocks()` (SUM of `slot_products.stock` grouped by `product_id`).

### 1.3 ⚠️ คำเตือน — อย่าใช้ API `remainingQuantity` แทน physical stock

จาก `AGENTS.md` (verbatim):

> Do **not** use `detail.remainingQuantity` from the API — that is issue-slip remaining, not physical stock.

- `remainingQuantity` = ยอดที่ **ใบเบิก (issue slip)** ยังเหลือให้หยิบ ไม่เกี่ยวกับว่าในช่องมีของกี่ชิ้น
- ใช้ผิด → จอแสดงว่ามีของทั้งที่ช่องว่าง หรือกลับกัน
- ของจริงเสมอ = `slotProductDao.getAllProductStocks()` (SUM ของ `slot_products.stock`)

---

## 2. กฎ sync ที่พังบ่อย

### 2.1 `picked_quantity` ถูกทับจาก delete-all + insert

จาก `AGENTS.md` (verbatim):

> **`picked_quantity` sync conflict:** `syncIssueSlips()` does delete-all + insert from API, which overwrites local `picked_quantity`. Always snapshot local values before sync and use `maxOf(local, remote)` when re-inserting. Same applies to `syncIssueSlipFromPayload()`.

**กลไกที่พัง:** `syncIssueSlips()` ลบของเดิมทั้งหมดแล้ว insert ใหม่จาก API → ถ้าตู้เพิ่งหยิบของไป (อัปเดต `picked_quantity` local แล้ว) แต่ API ยังไม่ทันรับรู้ → ค่าที่หยิบไปแล้วหายเกลี้ยง

**กฎที่ถูก:**
1. snapshot ค่า `picked_quantity` local ก่อน sync
2. ตอน re-insert ใช้ `maxOf(local, remote)` — ไม่ปล่อยให้ remote ที่เก่ากว่ามาทับ local
3. ใช้กับทั้ง `syncIssueSlips()` และ `syncIssueSlipFromPayload()`

### 2.2 Merged slots (`slotIdParent`)

จาก `AGENTS.md`:

> **Data model:** `SlotProductEntity.slotIdParent` links child slots to their primary/parent slot. Standalone slots have `slotIdParent = null`.

- หลาย slot ที่เป็นสินค้าเดียวกันถูก "merge" เป็นกลุ่ม โดย child slot ชี้ไป parent ผ่าน `slotIdParent` (standalone = `null`)
- **Display:** `RefillTabViewModel.loadProducts()` group ตาม `slotIdParent` → 1 merge group = 1 แถว ; `RefillProduct.stock` = **ผลรวม** stock ทุก slot ในกลุ่ม
- **Save (`onSaveRefill`):** re-query parent slot จาก DB เอา **actual** stock (ไม่ใช่ค่ารวมที่โชว์) → `stockDiff = refillAmount - actualParentStock` → อัปเดต **เฉพาะ record ของ parent** เท่านั้น
- **Tray badge:** หลัง save เรียก `refreshTrayBadges()` (ไม่ใช่ `loadTrays()` ซึ่งจะ reset tray ที่เลือก) ; badge query ใช้ merged-group logic (`GROUP BY slotIdParent`) เทียบ `SUM(stock)` กับ `minimum_stock`

> ฝั่ง API ฟิลด์ที่คุม merge คือ `slots.merge_role` + `slots.merged_with_slot_ids` (json) ; ฝั่งตู้ map เป็น `slotIdParent` ⚠️ verify mapping field-by-field กับโค้ด sync ถ้าต้องแก้ตรงนี้

---

## 3. Transaction sign convention (pickup / saveTransaction)

จาก `AGENTS.md` (verbatim):

> **`saveTransaction()`:** stores `amount = -product.amount` (negative for stock deduction). Must also set `requestAmount` (positive), `toolLife`, `reasonId`, `reasonDesc`, `kanbanId`, `kanbanName` for pickup history display.

**กฎเครื่องหมาย:**
- `amount` **ติดลบ** = ตัด stock (deduction). เก็บเป็น `-product.amount`
- ต้องเซ็ตฟิลด์เสริมให้ครบเพื่อให้ประวัติ pickup แสดงถูก:
  - `requestAmount` (เป็น**บวก** — จำนวนที่ขอ)
  - `toolLife`, `reasonId`, `reasonDesc`, `kanbanId`, `kanbanName`

**การแสดงผล:** จาก `AGENTS.md` —

> **`PickupHistoryRow`:** display `abs(txn.amount)` — never show raw negative values.

→ หน้าจอประวัติต้องโชว์ `abs(amount)` เสมอ ห้ามโชว์ค่าติดลบดิบ

**Mock mode** (`SerialService.USE_MOCK = true`):
- `reportPickup()` ข้าม API แต่ยังเรียก `updateLocalPickedQuantity()`
- `saveTransaction()` บันทึก record ด้วย `synced = true` และ `slotId = null`
- **`USE_MOCK` ต้องถูก guard ไว้เสมอ — release build ต้องวิ่งบน hardware จริง** (อย่าปล่อยติดไป production)

> ฝั่ง API ledger ที่ตรงกับ transaction ของตู้คือ `stock_movements` (มี `previous_quantity`/`new_quantity`/`tool_life`/`created_by_id`) — ส่ง user ผ่าน header `x-machine-user-id` ไม่งั้น `created_by_id` จะมั่ว (ดู `01-system-architecture.md`)

---

## 4. Minimum stock threshold

จาก `AGENTS.md` (verbatim):

> `minStockThreshold` on `MachineProductSlotResponseDto` comes from the API's `machine_products` table (machine-specific override).
> `ProductRepository.syncProductsFromPayload()` prefers `minStockThreshold` when > 0; falls back to `MIN(lowStockThreshold)` across slots for backward compatibility.

**ลำดับการเลือกค่า (precedence):**
1. ใช้ `minStockThreshold` (จาก `machine_products.min_stock_threshold` — override ราย**เครื่อง**) **เมื่อ > 0**
2. ถ้าไม่ (`= 0` / ไม่มี) → fallback เป็น `MIN(lowStockThreshold)` ข้าม slot (backward-compat)

เก็บผลลงเป็น `ProductEntity.minimumStock`

**ใช้ที่ไหน:**
- **Tray badge** — query นับสินค้าที่ต่ำกว่า minimum ต่อ tray (`countBelowMinimumPerTray`, merged-group logic, `SUM(stock)` vs `minimum_stock`)
- **Refill UI** — highlight สินค้าที่ stock ต่ำกว่า minimum

**ตารางฝั่ง API ที่เกี่ยว:**
- `machine_products.min_stock_threshold` — override ราย machine
- `products.min_stock_threshold` + `products.standard_stock` — ค่า default ระดับสินค้า
- `slots.low_stock_threshold` — เกณฑ์ราย slot (ฐานของ fallback `MIN(...)`)

> ⚠️ verify: ชื่อ field ฝั่ง API เป็น snake_case (`min_stock_threshold`) ส่วนฝั่ง DTO/ตู้เป็น camelCase (`minStockThreshold`) — เป็นฟิลด์เดียวกัน

---

## 5. โมเดลอนาคต — Return Stock / Tool Room (Phase 1–3)

> ยังไม่ build ใน release ปัจจุบัน — สรุปจาก PRD เพื่อให้ทีมเข้าใจทิศทางก่อนเริ่ม
> เอกสารแม่ + design ของจริง: ดู PRD ในชุดเอกสาร dev (`PRD-return-stock-toolroom.md`, `dbdiagram-return-stock.dbml`, `design-tradeoffs-return-stock.md`)

### 5.1 จาก 1 ก้อน → 3 ก้อนต่อสินค้า

วันนี้ stock มีก้อนเดียว (ในตู้/slot) อนาคตแยกเป็น **3 pool ต่อสินค้า**:

| Pool | คืออะไร | เกิดใน |
|---|---|---|
| **Vending** | ของในตู้ vending จริง (slot stock เดิม) | มีอยู่แล้ว |
| **Tool Room** | ของในห้องเครื่องมือ (ตู้เสมือน ไม่มี hardware) | Phase 2 |
| **Return** | ของคืนสะสมรอส่งลับคม (regrind) | Phase 1 (pool) → Phase 2 (เป็นเครื่อง type `return`) |

หน้าคลังสินค้า (Phase 2) จะแสดง **ยอดรวมต่อสินค้า + breakdown** ว่าอยู่ก้อนไหนเท่าไหร่ → ครั้งแรกที่ระบบมีแนวคิด "คลัง" เป็นมิติแยกจาก slot

**Design ที่เลือก:** ขยาย `machines` ให้มี machine-type (`vending` / `toolroom` / `return`) ตั้งแต่ Phase 1 เพื่อไม่ rework — และ **reuse `stock_movements` engine เดิมทั้งหมด** โดยเพิ่ม dimension ของ pool ต้นทาง/ปลายทาง (ไม่ fork ledger ใหม่)

### 5.2 Move stock ต้องผ่านเอกสารเสมอ (ใบคืน / ใบเบิก)

ทุกการย้ายของระหว่างก้อนเกิดผ่านเอกสาร ไม่มีการ set stock ลอย ๆ:

- **ใบคืน (return slip)** — Phase 1. ตารางใหม่ `return_slips` / `return_slip_items` / `return_slip_collections` (clone แนวทาง issue-slip เดิม) ; พนักงานคืนที่ตู้ → เจ้าหน้าที่ยืนยันจำนวนจริง → ยอดเข้า Return pool
- **ใบเบิกเติมตู้ (Tool Room → Vending)** — Phase 2. ตัด Tool Room ทันที ไม่ต้องอนุมัติ
- **ใบเบิกใช้งาน** — Phase 2/3. ตัด Tool Room เมื่อ approve ครบทุกขั้น (Approval Policy)
- **ใบส่ง regrind + Receive (partial)** — Phase 2. ตัด Return ออกไปลับคม (ผูกเลข PO) แล้วรับกลับเข้า Tool Room ทีละงวด
- **การเปิด locker นอก flow เบิก** = **device command** ตรง ("เปิด bin N") **แยกขาดจาก stock logic** — ไม่ set stock หลอก (PoC เป็น gate ก่อนเริ่ม Phase 1)

### 5.3 Unit conversion (UoM) — กก. ↔ ชิ้น ↔ เซ็ต

Phase 2 เพิ่มหน่วยนับหลายแบบ + ตัวคูณแปลงต่อสินค้า:

- ตารางใหม่ `product_uom` (`unit_code`, `conversion_factor` → หน่วยฐาน = ชิ้น ; เช่น 1 เซ็ต = 10 ชิ้น)
- ⚠️ `units` table เดิมใน `schema.dbml` (`units.pieces`) เป็น org/UoM ระดับเบื้องต้น — PRD ระบุว่า UoM ของจริงต้องสร้าง `product_uom` ใหม่ ไม่ reuse `units` ตรง ๆ
- เลือกหน่วยตอนทำใบเบิก/รับของ → ระบบแปลงเป็นหน่วยฐาน ; รายงานทุกหน้าเป็นหน่วยฐานเดียวกัน
- ขนาดเซ็ตของ Phase 1 (เก็บง่าย ๆ ต่อสินค้า) จะ migrate เข้า `product_uom` ใน Phase 2

### 5.4 Adjust stock

- ลดยอดก้อนใด ๆ ได้ + **บังคับใส่เหตุผล** + บันทึกผู้ทำ (ใช้กลไก `stock_movements` + audit log เดิม)
- Phase 1: adjust Return pool เป็น escape hatch ชั่วคราว (ตัดของที่ส่งไปลับคม ระหว่างรอ flow regrind เต็มรูปใน Phase 2)

### 5.5 Borrow-limit ใหม่ — "คืนก่อนถึงเบิกใหม่ได้"

เปลี่ยนกติกาสิทธิ์เบิกจาก "โควต้าต่อวัน (reset เอง)" → **checkout ledger ต่อ `(user × product)`**:

- ตารางใหม่ `checkout_balances` (`issued_qty` / `returned_qty` / `outstanding_qty = issued − returned` / `set_size`) — เป็น projection ที่ derive ได้จาก issue/return movement
- ปลดล็อกสิทธิ์ = `floor(returned_qty / set_size)` เซ็ต — คืนไม่ครบเซ็ตยังเบิกชุดใหม่ไม่ได้ แต่เศษที่คืนถูกสะสมไว้ (คืน 15/20 → ปลด 1 เซ็ต เศษ 5 รอ)
- enforcement แยก sub-release ได้ (วงจรคืนทำงานก่อน เปิดบังคับทีหลัง เพราะแตะ flow เบิกบน production)

### 5.6 ลิงก์ PRD แต่ละเฟส

| Phase | ขอบเขต | PRD (ดู PRD ในชุดเอกสาร dev) |
|---|---|---|
| Phase 1 | คืนผ่าน locker + Return pool + checkout ledger + adjust | `PRD-phase1-return-via-locker.md` |
| Phase 2 | stock 3 ก้อน + ตู้เสมือน + ใบเบิก + UoM + regrind | `PRD-phase2-toolroom-issue-approval.md` |
| Phase 3 | Approval Policy + Position/Level + จัดซื้อ/quotation email | `PRD-phase3-procurement.md` |
| สรุปแบ่งเฟส | feature ต่อ phase + ราคา | `features-by-phase-return-stock.md` |

---

## 6. ตาราง Pitfall → กฎที่ถูก

| # | สิ่งที่ทำพังบ่อย (pitfall) | กฎที่ถูก |
|---|---|---|
| 1 | ใช้ API `remainingQuantity` เป็น physical stock | ใช้ `slotProductDao.getAllProductStocks()` = `SUM(slot_products.stock)` group by `product_id` |
| 2 | sync ใบเบิก (delete-all + insert) ทับ `picked_quantity` local ที่เพิ่งหยิบ | snapshot ก่อน sync + ใช้ `maxOf(local, remote)` ตอน re-insert (ทั้ง `syncIssueSlips()` + `syncIssueSlipFromPayload()`) |
| 3 | merged slot: เอายอดรวมที่โชว์ไปคิด stockDiff / อัปเดตทุก slot | re-query parent เอา actual stock → `stockDiff = refillAmount - actualParentStock` → อัปเดต **เฉพาะ parent** |
| 4 | refill เสร็จเรียก `loadTrays()` ทำให้ tray ที่เลือก reset | เรียก `refreshTrayBadges()` แทน |
| 5 | เก็บ pickup `amount` เป็นบวก / โชว์ค่าติดลบดิบ | เก็บ `amount = -product.amount` (ลบ = ตัด) ; display `abs(amount)` เสมอ |
| 6 | ลืมเซ็ต `requestAmount`/`toolLife`/`reasonId`/`kanbanId` ตอน `saveTransaction()` | เซ็ตให้ครบ ไม่งั้นประวัติ pickup แสดงไม่ครบ |
| 7 | ปล่อย `SerialService.USE_MOCK = true` ติดไป release | guard ไว้เสมอ — release ต้องวิ่ง hardware จริง |
| 8 | ใช้ `minStockThreshold` ทั้งที่เป็น 0 | ใช้ `minStockThreshold` **เมื่อ > 0** ก่อน ไม่งั้น fallback `MIN(lowStockThreshold)` |
| 9 | ลืมส่ง header `x-machine-user-id` → `created_by_id` / ผู้ทำ movement มั่ว | ส่ง header ทุก request ของตู้ (ดู `01-system-architecture.md`) |
| 10 | (อนาคต) set stock หลอกตอนเปิด locker / ย้ายของโดยไม่มีเอกสาร | เปิด locker = device command แยกจาก stock ; ทุก move ต้องผ่านใบคืน/ใบเบิก + `stock_movements` |

---

## อ้างอิง source

- `vending-machine-kotlin/AGENTS.md` — sections: *Delivery & Pickup Flow*, *Merged Slots*, *Minimum Stock Threshold*, *Machine Auth Header*
- `vending-machine-api/docs/schema.dbml` — `slots`, `stocks`, `stock_movements`, `machine_products`, `products`, `issue_slip_*`, `units`
- `vending-machine-api/docs/er-diagram.mmd` — ER diagram (ภาพรวม FK)
- PRD Return Stock (ดู PRD ในชุดเอกสาร dev): `PRD-phase1-return-via-locker.md`, `PRD-phase2-toolroom-issue-approval.md`, `PRD-phase3-procurement.md`, `features-by-phase-return-stock.md`, `dbdiagram-return-stock.dbml`
- เอกสารพี่น้อง: [01-system-architecture.md](01-system-architecture.md) · [02-debugging-runbook.md](02-debugging-runbook.md) · แผนอบรม internal (ดูชุดเอกสาร dev)
