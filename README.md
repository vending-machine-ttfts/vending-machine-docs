# เอกสารระบบ Vending Machine

ชุดเอกสารสำหรับ **ติดตั้ง ดูแล และใช้งาน** ระบบตู้จำหน่ายสินค้าอัตโนมัติ (Vending Machine) — ครอบคลุมตั้งแต่เตรียมเครื่อง ติดตั้ง server/ตู้ ไปจนถึง debug และ deploy

ระบบประกอบด้วย 5 ส่วน: **Web Admin** (จัดการระบบ) · **API** (backend) · **Database** · **ตู้ Android** (หน้าตู้ kiosk) · **Updater** (อัปเดตแอปตู้)

---

## เอกสารในชุดนี้

### สำหรับทีมเทคนิค (ติดตั้ง / ดูแลระบบ)

| เอกสาร | เนื้อหา |
|---|---|
| [pre-install-checklist.md](pre-install-checklist.md) | Checklist โปรแกรมที่ต้องติดตั้งล่วงหน้า — server, เครื่อง dev, ตู้ |
| [customer-pre-install-guide.md](customer-pre-install-guide.md) | รายการโปรแกรม + วิธีติดตั้งทีละขั้น (Windows) |
| [tech-team-handbook.md](tech-team-handbook.md) | **คู่มือหลัก** — setup server/ตู้, Tailscale, โครงสร้างระบบ, debug, การคำนวณ stock, deploy/update (เคส offline) |

> เอกสารเพิ่มเติมที่จะตามมา: `server-setup-guide.md` (setup server เต็ม), `training-plan.md` (อบรมผู้ใช้งาน), `full-manual/` (คู่มือฟีเจอร์ทุกหน้าจอ)

---

## เริ่มที่ไหน

**ก่อนวันติดตั้ง (เตรียมเครื่อง):**
1. อ่าน [pre-install-checklist.md](pre-install-checklist.md) — รายการโปรแกรมที่ต้องลงรอไว้
2. ทำตาม [customer-pre-install-guide.md](customer-pre-install-guide.md) — ติดตั้งให้ครบก่อนวันจริง

**วันติดตั้ง + ดูแลต่อ:**
- ใช้ [tech-team-handbook.md](tech-team-handbook.md) เป็นหลัก — จัดเรียงตามลำดับงานจริง: Server → ตู้ Android + Tailscale → เพิ่มเครื่องในระบบ → debug → stock → deploy

---

## หมายเหตุ

- เอกสารชุดนี้เป็น **ฉบับส่งมอบ** — ตัวเลข พอร์ต โดเมน และชื่อ brand บางจุดเป็นตัวอย่าง ให้ปรับตามค่าจริงของหน้างาน
- เครื่องมือ remote ที่แนะนำ: **Tailscale + scrcpy** (เข้าตู้/server ได้โดยไม่ต้อง config router)
