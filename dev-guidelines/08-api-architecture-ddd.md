# 08 — API Architecture (DDD ลงลึก)

> เอกสารอ้างอิงสำหรับนักพัฒนา — เจาะ **architecture ของ `vending-machine-api`** ระดับโค้ด
> ขยายต่อจาก §7 ของ [01-system-architecture.md](01-system-architecture.md) (ภาพรวม) — ที่นี่ลงลึก **DDD tactical patterns** + กฎที่ต้องรักษา
> ทุกตัวอย่างในเอกสารนี้ verify จาก source จริง (module `issue-slip`)

---

## 1. หลักการ (สรุป 1 บรรทัด)

**Domain-Driven Design (tactical) + Clean / Hexagonal Architecture (Ports & Adapters) — 1 bounded context ต่อ 1 feature module**

ไม่ใช่ NestJS แบบ default (controller → service → entity) — **domain core ไม่รู้จัก NestJS / TypeORM / HTTP เลย**

ขนาดปัจจุบัน (รู้ scale ที่กำลังทำงานด้วย):

| สิ่งที่นับ | จำนวน |
|---|---|
| Feature modules (bounded contexts) | 26 |
| Use-cases | ~254 |
| Domain repository ports (interface) | 44 |
| Value objects | 74 |
| ORM entities | 59 |

---

## 2. โครงสร้าง 4 ชั้น

ทุก module ใต้ `src/modules/<feature>/` มี folder ชุดเดียวกัน:

```
modules/issue-slip/
├── domain/            ← business core. ห้าม import framework
│   ├── entities/         domain model จริง (issue-slip.entity.ts)
│   ├── value-objects/    typed ID + value (issue-slip-id.vo.ts, issue-slip-status.vo.ts)
│   ├── repositories/     PORTS — interface ล้วน (issue-slip.repository.interface.ts)
│   ├── exceptions/       domain error (IssueSlipNotFoundException…)
│   └── constants/
├── application/       ← orchestration. รู้จัก domain ไม่รู้จัก infrastructure
│   ├── use-cases/        1 class = 1 action (get-issue-slip.use-case.ts)
│   └── dto/              request/response shape
├── infrastructure/    ← ADAPTERS. ชั้นเดียวที่แตะ TypeORM
│   ├── orm/              TypeORM @Entity (*.orm-entity.ts)
│   └── repositories/     impl ของ port (typeorm-issue-slip.repository.ts)
├── presentation/      ← guard, decorator เฉพาะ transport ของ module
└── issue-slip.module.ts  ← DI wiring
```

base class กลางอยู่ที่ `src/shared/domain/`:
`entity.base.ts` · `aggregate-root.base.ts` · `value-object.base.ts` · `identifier.ts`

> **Controller ไม่ได้อยู่ใน module** — อยู่ที่ `src/api/` แยกตาม audience (ดู §7)

---

## 3. กฎ Dependency (หัวใจของ pattern)

dependency ชี้ **เข้าด้านในทางเดียว**:

```
presentation ─► application ─► domain ◄─ infrastructure
                                  ▲
                       (infra implement port ของ domain)
```

- `domain` ไม่ import อะไรออกข้างนอก — ไม่มี `@nestjs/*`, ไม่มี `typeorm`, ไม่มี DTO
- `application` import `domain` (entity, VO, port interface) — **ห้าม** import `infrastructure`
- `infrastructure` import `domain` เพื่อ **implement** interface และ map กับ ORM
- module (wiring) เป็นที่เดียวที่รู้จักครบทั้ง 3 ชั้น

ถ้าเมื่อไหร่ use-case ไป `import` ไฟล์ `typeorm-*.repository.ts` หรือ `*.orm-entity.ts` → **ผิดกฎ** ให้ใช้ผ่าน port แทน

---

## 4. Request flow

```
HTTP / Socket
   │
   ▼
Controller (src/api/...)         ← บาง: validate DTO → เรียก use-case → คืน DTO
   │  inject UseCase
   ▼
UseCase (application)            ← orchestrate: โหลด aggregate ผ่าน PORT, รัน domain logic, persist
   │  inject IRepository (port, ผ่าน Symbol token)
   ▼
TypeOrm*Repository (infra)       ← implement port, map Domain ⇆ OrmEntity
   │
   ▼
TypeORM → SQL Server
```

use-case พึ่ง `IIssueSlipRepository` (interface) — ตอน runtime DI สลับ `TypeOrmIssueSlipRepository` เข้าให้
use-case ไม่รู้ว่าคุยกับ DB ตัวไหน → **mock ใน unit test ได้ง่าย**

---

## 5. DDD building blocks (โค้ดจริง)

### Value Object — identity มี type, ไม่ใช่ string ลอย ๆ
`domain/value-objects/issue-slip-id.vo.ts`
```ts
export class IssueSlipId extends UniqueEntityID {
  private constructor(id?: string) { super(id) }
  public static create(id?: string): IssueSlipId { return new IssueSlipId(id) }
}
```
`UniqueEntityID` (`shared/domain/identifier.ts`) สร้าง **uuidv7** อัตโนมัติเมื่อไม่ส่ง id เข้ามา และให้ `.equals()` / `.toValue()`
→ `IssueSlipId` กับ `UserId` เป็น **คนละ type** ส่งสลับกันไม่ได้ (กัน bug ส่ง id ผิดตัว)

> เกี่ยวโยง: header `x-machine-user-id` มีไว้เพราะถ้าไม่มี user จริง `getCurrentUserId()` จะสุ่ม uuidv7 ใหม่ทุกครั้ง → pickup จะ unauthorized

### Port — interface อยู่ใน `domain` + DI token เป็น Symbol
`domain/repositories/issue-slip.repository.interface.ts`
```ts
export const ISSUE_SLIP_REPOSITORY = Symbol('ISSUE_SLIP_REPOSITORY')

export interface IIssueSlipRepository {
  save(issueSlip: IssueSlip): Promise<void>
  saveTx(em: EntityManager, issueSlip: IssueSlip): Promise<void>
  findById(id: IssueSlipId): Promise<IssueSlip>
  findByIdWithItems(id: IssueSlipId): Promise<IssueSlip>
  // … คืน DOMAIN entity, รับ VO
}
```
port พูดภาษา **domain** (คืน `IssueSlip`, รับ `IssueSlipId`) ไม่ใช่ ORM row
`Symbol` คือ DI token — TS interface หายตอน runtime จึง inject by type ไม่ได้ ต้อง inject by token

### Use-case — 1 action, พึ่งแค่ port
`application/use-cases/get-issue-slip.use-case.ts`
```ts
@Injectable()
export class GetIssueSlipUseCase {
  constructor(
    @Inject(ISSUE_SLIP_REPOSITORY) private readonly issueSlipRepository: IIssueSlipRepository,
    @Inject(MACHINE_REPOSITORY)    private readonly machineRepository: IMachineRepository,
    // … port อื่น + use-case ข้าม module
  ) {}

  async execute(issueSlipId: string): Promise<IssueSlipResponseDto> {
    const id = IssueSlipId.create(issueSlipId)
    const issueSlip = await this.issueSlipRepository.findByIdWithItems(id)
    if (!issueSlip) throw new IssueSlipNotFoundException(issueSlipId)
    // …ประกอบ response จาก aggregate ที่โหลดผ่าน port
  }
}
```
pattern: **1 class, 1 `execute()`, 1 business action**
ข้าม module → ผ่าน port หรือ use-case ที่ module นั้น **export** (เช่น `IMachineRepository`) — ห้ามยิง table ของ context อื่นตรง ๆ

### Adapter — โค้ดเดียวที่รู้จัก TypeORM
`infrastructure/repositories/typeorm-issue-slip.repository.ts`
```ts
@Injectable()
export class TypeOrmIssueSlipRepository implements IIssueSlipRepository {
  constructor(
    @InjectRepository(IssueSlipOrmEntity) private readonly issueSlipRepository: Repository<IssueSlipOrmEntity>,
    // …
  ) {}

  async save(issueSlip: IssueSlip): Promise<void> {
    return this.saveTx(this.issueSlipRepository.manager, issueSlip)
  }

  private toOrmEntity(d: IssueSlip): IssueSlipOrmEntity { /* domain → row */ }
  private toDomain(o: IssueSlipOrmEntity): IssueSlip   { /* row → domain */ }
}
```
adapter `implements` port และเป็นเจ้าของ **mapping Domain ⇆ ORM**
domain entity (`IssueSlip`) กับ ORM entity (`IssueSlipOrmEntity`) เป็นคนละ class **ตั้งใจ** — shape สำหรับ persist ≠ shape สำหรับ business

### Transaction
port มี variant `saveTx(em, …)` / `getNextIssueNoTx(em, …)` ที่รับ `EntityManager`
→ use-case รันหลาย repo ใน transaction เดียวได้ โดย domain ไม่ต้องรู้ว่า transaction คืออะไร

---

## 6. Module wiring (จุดที่ผูก abstraction เข้ากับ concrete)

`issue-slip.module.ts`
```ts
@Module({
  imports: [ TypeOrmModule.forFeature([ IssueSlipOrmEntity, /* … */ ]), MachineModule, ProductModule, /* … */ ],
  providers: [
    GetIssueSlipUseCase, CreateIssueSlipUseCase, /* … use-case ทั้งหมด */,
    { provide: ISSUE_SLIP_REPOSITORY, useClass: TypeOrmIssueSlipRepository },   // ← port → adapter
  ],
  exports: [ ISSUE_SLIP_REPOSITORY, GetIssueSlipUseCase, /* ที่ module อื่นใช้ได้ */ ],
})
export class IssueSlipModule {}
```
- `{ provide: TOKEN, useClass: Impl }` = **จุดที่ abstraction เจอ concrete** เปลี่ยน `useClass` = เปลี่ยน DB tech โดย use-case ไม่ต้องแก้
- module **export** เฉพาะ port + use-case ที่ตั้งใจให้ใช้ซ้ำ → bounded context อื่นยัง decouple กัน

---

## 7. Transport split (controller)

controller อยู่ที่ `src/api/` จัดกลุ่ม **ตาม audience ไม่ใช่ตาม feature**:

```
src/api/
├── portal/    ← Web admin (vending-machine-web) — CRUD เต็ม, JWT auth (33 controllers)
└── machine/   ← Kiosk (vending-machine-kotlin) — api-key auth, scope ต่อเครื่อง
```

`machine.controller.ts` (portal) กับ endpoint ฝั่ง kiosk เป็นคนละ transport บน **use-case ชุดเดียวกัน**
`presentation/` ระดับ module เก็บ guard/decorator ของ transport (เช่น `api-key.guard.ts`, `require-machine-user.decorator.ts`)
controller ต้อง **บาง**: parse/validate DTO → เรียก use-case → คืน DTO — **ห้าม** มี business logic

---

## 8. Recipe — เพิ่ม feature ให้ถูก pattern

ตัวอย่าง "approve issue slip":

1. **Domain ก่อน** — มี state/กฎใหม่? แก้ที่ `IssueSlip` entity / `IssueSlipStatus` VO. ถ้ากฎ fail ได้ ให้เพิ่ม domain exception
2. **Port** — ต้อง query/persist แบบใหม่? เพิ่ม method ใน `IIssueSlipRepository` (+ variant `…Tx` ถ้าเข้าร่วม transaction)
3. **Adapter** — implement method นั้นใน `TypeOrmIssueSlipRepository`, map ORM ⇆ domain. **ที่เดียว**ที่เขียน SQL/TypeORM
4. **Use-case** — สร้าง `ApproveIssueSlipUseCase` มี `execute()`. inject port, orchestrate, คืน DTO
5. **Wire** — เพิ่ม use-case ใน `providers` (+ `exports` ถ้าใช้ซ้ำ) ของ module
6. **Transport** — เพิ่ม controller method ใน `src/api/portal` หรือ `src/api/machine` ที่เรียก use-case
7. **Test** — unit test use-case ด้วย port ที่ mock (ไม่ต่อ DB). VO/entity มี `.spec.ts` ของตัวเอง (ดู `issue-slip-id.vo.spec.ts`)

เพิ่ม entity / แก้ schema → ทำ TypeORM migration (`pnpm migration:generate`) **ห้าม** auto-sync

---

## 9. กฎ Do / Don't

**Do**
- `domain/` ต้องปลอด framework — ถ้า import `@nestjs` หรือ `typeorm` = วางผิดชั้น
- inject repository **ผ่าน Symbol token**, type เป็น **interface**
- คืน/รับ **VO + domain entity** ข้าม port — ไม่ใช่ ORM row, ไม่ใช่ string ดิบ
- 1 use-case = 1 action
- ข้าม module = import module นั้น แล้ว inject port/use-case ที่มัน **export** — ห้ามยิง table ของ context อื่น
- map ชัด ๆ ใน adapter (`toDomain` / `toOrmEntity`)

**Don't**
- ห้าม inject `Repository<…OrmEntity>` เข้า use-case
- ห้าม import `infrastructure/*` จาก `application/*` หรือ `domain/*`
- ห้ามมี business logic ใน controller หรือใน ORM entity
- ห้ามส่ง `…OrmEntity` ออกนอก `infrastructure/` (ยกเว้น `findPaginated` คืน `Paginated<…OrmEntity>` สำหรับ endpoint list — shortcut ตั้งใจของ `nestjs-paginate`)
- ห้ามใช้ VO id ปนข้าม context (`IssueSlipId` ≠ `UserId`)

---

## 10. Cheat sheet — เขียน X ไว้ที่ไหน

| กำลังเขียน… | วางที่ |
|---|---|
| กฎธุรกิจ / invariant | `domain/entities` หรือ `domain/value-objects` |
| typed ID / value | `domain/value-objects/*.vo.ts` |
| "ต้องโหลด/เซฟ X" (สัญญา) | `domain/repositories/*.interface.ts` (port) |
| domain error | `domain/exceptions` |
| action / workflow ของ app | `application/use-cases/*.use-case.ts` |
| request/response shape | `application/dto` |
| TypeORM `@Entity` | `infrastructure/orm/*.orm-entity.ts` |
| SQL / query / mapping จริง | `infrastructure/repositories/typeorm-*.repository.ts` |
| HTTP/socket endpoint | `src/api/portal` หรือ `src/api/machine` |
| DI binding `token → impl` | `<feature>.module.ts` providers |

---

*ตัวอย่างมาตรฐานให้ลอกแบบ: module `issue-slip` — ครบทุกชั้น (VO, aggregate + items, transactional port, use-case ข้าม module, ทั้ง 2 transport)*
