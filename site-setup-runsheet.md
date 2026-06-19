# Site Setup Runsheet (on-site) ✅ เช็กลิสต์กดตามจริง

> ทฤษฎี/รายละเอียดทุก step → [`tech-team-handbook.md` §1](tech-team-handbook.md) · ตู้ Android → §2 · พังหน้างาน → ภาคผนวก B (Troubleshooting รวม)
> ค่าตัวอย่าง: brand=`denso`, ports `9080/9081/9082/9083`, domain `denso.local`
> 🖱️ **วิธีง่ายสุด:** ดับเบิลคลิก **`C:\gitops-bootstrap\0-START-HERE.cmd`** อันเดียว → เมนูเลข 1-7 (ตรงกับ Phase) → ไม่ต้องเปิดไฟล์ `.ps1` เอง. คำสั่งมือด้านล่างไว้เผื่อ debug

## ค่าที่ต้องมีในมือก่อนเริ่ม (เติมก่อน)
- [ ] **PAT** (scope `repo` / Contents:Read — ดู §1.2) = `__________`
- [ ] **sa password** = `__________`
- [ ] **redis password** = `__________`
- [ ] **Docker Hub** user/token = `__________`
- [ ] **domain** = `denso.local` (หรือ `________`)
- [ ] machine name (`DB_HOST`) = หาในเครื่องด้วย `$env:COMPUTERNAME`
- [ ] branding (โลโก้/สี)

---

## Phase 0 — Pre-flight
- [ ] TeamViewer **unattended access** ตั้งแล้ว (เผื่อ reboot ต่อกลับได้)
- [ ] pre-staged ไว้? (ถ้า teardown `-KeepSecrets` แล้ว → base+image warm, ข้าม Phase 2 ได้ — git+repo มีแล้ว)
- [ ] ล้าง site เก่าเพื่อ re-demo → 🖱️ **ดับเบิลคลิก `C:\gitops-bootstrap\teardown-site.cmd`** → checkbox เลือก site (Space=toggle, Enter), พิมพ์ `yes` ยืนยัน (เก็บ DB + images; จะ drop DB ต้องรัน `.ps1 -DropDatabase` เอง). teardown คืน port ใน registry ให้อัตโนมัติ (เก็บ DB + images; จะ drop DB ต้องรัน `.ps1 -DropDatabase` เอง). teardown คืน port ใน registry ให้อัตโนมัติ

## Phase 1 — Bootstrap: git + auth + clone  (paste ล้วน ไม่ต้องโยนไฟล์)
- [ ] มี PAT แล้ว? (ไม่มี → github.com/settings/personal-access-tokens → fine-grained → resource owner=org → repo `vending-machine-gitops` → Contents:Read → copy)
- [ ] เปิด **Windows PowerShell → Run as Administrator**
- [ ] **install git** (ไม่ต้อง winget; TLS1.2 + silent; ข้ามถ้ามีแล้ว):
  ```powershell
  if (-not (Get-Command git -ErrorAction SilentlyContinue)) {
    [Net.ServicePointManager]::SecurityProtocol=[Net.SecurityProtocolType]::Tls12
    $v="2.47.1"; $o="$env:TEMP\git.exe"
    Invoke-WebRequest "https://github.com/git-for-windows/git/releases/download/v$v.windows.1/Git-$v-64-bit.exe" -OutFile $o
    Start-Process $o -Wait -ArgumentList '/VERYSILENT','/NORESTART','/SP-','/SUPPRESSMSGBOXES'
    $env:Path="C:\Program Files\Git\cmd;$env:Path"
  }
  git --version   # ยืนยันมี git (เวอร์ชันใหม่กว่าได้ที่ git-scm.com/download/win)
  ```
- [ ] **git auth** (เก็บ PAT ใน Windows Credential Manager — ไม่ค้างใน URL/ไฟล์):
  ```powershell
  git config --global credential.helper manager
  $sec=Read-Host "GitHub PAT" -AsSecureString
  $pat=[Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($sec))
  "protocol=https`nhost=github.com`nusername=x-access-token`npassword=$pat" | git credential approve
  git ls-remote https://github.com/vending-machine-ttfts/vending-machine-gitops.git HEAD   # ไม่ถาม = OK
  ```
- [ ] **clone** (URL ปกติ ไม่ใส่ PAT — GCM จัดให้):
  ```powershell
  git clone https://github.com/vending-machine-ttfts/vending-machine-gitops.git C:\gitops-bootstrap
  cd C:\gitops-bootstrap
  ```
  - จากนี้ทุกอย่างรันจาก repo → 🖱️ **`C:\gitops-bootstrap\0-START-HERE.cmd`** (เมนูเลขตาม Phase). clone ต่อ site ก็ silent (token แคชแล้ว)

## Phase 2 — Base tooling (Docker / NSSM / IIS)  (จาก repo)
- [ ] 🖱️ ดับเบิลคลิก **`C:\gitops-bootstrap\0-START-HERE.cmd`** → เลือก **1) Base tooling** (= setup-server, tee `C:\setup-server.log`)
  - หรือมือ: `powershell -ExecutionPolicy Bypass -File C:\gitops-bootstrap\setup-server.ps1 *>&1 | Tee-Object C:\setup-server.log`
- [ ] เจอ `REBOOT REQUIRED`? → `Restart-Computer` → reconnect → ดับเบิลคลิก `0-START-HERE.cmd` → ข้อ 1 ซ้ำ (idempotent)
- [ ] verify:
  ```powershell
  docker version            # Server = windows/amd64
  docker network ls         # มี gitops_default (nat)
  Get-WindowsFeature Web-Server | Select InstallState   # Installed
  ```
  - nssm.cc 503? → patched: mirror auto-fallback (ดู §1.1 Gotchas)

## Phase 3 — Services + ยืนยัน DB
- [ ] Redis + Mailpit — 🖱️ **ดับเบิลคลิก `C:\gitops-bootstrap\setup-windows-services.cmd`** (default ครบ, redis pwd ตรง init-secrets) — หรือมือ (custom redis pwd):
  ```powershell
  pwsh -File .\setup-windows-services.ps1 -RedisPassword "<redis-pwd>" -NssmPath C:\nssm\win64\nssm.exe
  ```
- [ ] **เช็ค Redis** (service + ping + port):
  ```powershell
  Get-Service *redis*                                              # Status = Running
  C:\services\redis\redis-cli.exe -a "<redis-pwd>" ping            # → PONG (ต่อ + auth ผ่าน)
  Test-NetConnection localhost -Port 6379 | Select TcpTestSucceeded  # True
  ```
  - Mailpit: `Test-NetConnection localhost -Port 8025 | Select TcpTestSucceeded` → True (UI: http://localhost:8025)
- [ ] **SQL Server (Express, instance `SQLEXPRESS` บนเครื่อง `VENDINGSRV`)** — เช็ก + เตรียมให้ container ต่อได้:
  ```powershell
  # auto-detect instance (Standard default MSSQLSERVER / Express named) — Denso = localhost\SQLEXPRESS
  $inst = if (Get-Service MSSQLSERVER -EA SilentlyContinue) {'localhost'} else {'localhost\' + ((Get-Service 'MSSQL$*' | Select -First 1).Name -split '\$')[1]}
  Get-Service 'MSSQL*' | Where Status -eq 'Running' | Select Name
  sqlcmd -S $inst -E -C -Q "SELECT SERVERPROPERTY('Edition') Edition, SERVERPROPERTY('InstanceName') Instance;"   # -C = ODBC18 trust self-signed cert
  ```
  - หมายเหตุ: Mixed Mode + TCP step ด้านล่าง = **เฉพาะ Express named instance** (ถ้า Standard default `MSSQLSERVER` → listen 1433 + sa อยู่แล้ว มักข้าม TCP step ได้)
  - [ ] **เปิด Mixed Mode + sa** (SSMS New Query, Windows auth — Express default ปิด sa ไว้):
    ```sql
    EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', N'Software\Microsoft\MSSQLServer\MSSQLServer', N'LoginMode', REG_DWORD, 2;
    ALTER LOGIN sa ENABLE; ALTER LOGIN sa WITH PASSWORD = '<sa-pwd>';
    ```
    - ⭐ **Mixed Mode + sa มีผลหลัง restart เท่านั้น** → restart ในขั้นถัดไปคุมทั้งคู่ (อย่า verify sa ก่อน restart = `Login failed for sa`)
  - [ ] **เปิด TCP static 1433** (Express named instance = dynamic port; API ต่อ host:1433) แล้ว restart:
    ```powershell
    $t = "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\$((Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL').SQLEXPRESS)\MSSQLServer\SuperSocketNetLib\Tcp"
    Set-ItemProperty $t -Name Enabled -Value 1
    Set-ItemProperty "$t\IPAll" -Name TcpDynamicPorts -Value ''; Set-ItemProperty "$t\IPAll" -Name TcpPort -Value '1433'
    Restart-Service 'MSSQL$SQLEXPRESS' -Force
    ```
  - [ ] verify (หลัง restart — `-C` ทุก sqlcmd, ODBC18 trust cert):
    ```powershell
    sqlcmd -S $inst          -U sa -P "<sa-pwd>" -C -Q "SELECT @@VERSION;"   # sa login (instance, $inst จาก check ด้านบน) — ทั้ง 2 edition
    sqlcmd -S localhost,1433 -U sa -P "<sa-pwd>" -C -Q "SELECT 1"            # TCP 1433 = path ที่ container ใช้ — ทั้ง 2 edition
    Test-NetConnection localhost -Port 1433 | Select TcpTestSucceeded             # True
    ```
  - [ ] **DB_HOST = `VENDINGSRV`** (host:1433) ; setup-site ใช้ `-SqlServerInstance localhost\SQLEXPRESS`
  - ✅ `doctor.ps1` detect Express (`MSSQL$SQLEXPRESS`) แล้ว — ไม่ false-fail แล้ว
- [ ] firewall:
  ```powershell
  New-NetFirewallRule -DisplayName "SQL 1433 in" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
  # (+ 6379 redis, 1025 mailpit ถ้า container ต้องต่อ)
  ```

## Phase 4 — ขึ้น site (สาธิตสด)
- [ ] สร้าง site — 🖱️ **ดับเบิลคลิก `C:\gitops-bootstrap\setup-site.cmd`** → ถาม brand/domain/ports ทีละช่อง (Enter = default), จอง port ให้เอง. denso ใส่ ports 9080-9083 / redisDb 2 ตอนถาม:
  ```powershell
  # หรือมือ: domain (denso.local), DB (Vending_Denso), SQL instance = auto-detect — ใส่แค่ ports (non-default)
  powershell -ExecutionPolicy Bypass -File .\setup-site.ps1 -Brand denso -CreateDatabase `
    -BlueWebPort 9080 -BlueApiPort 9081 -GreenWebPort 9082 -GreenApiPort 9083 -RedisDb 2
  ```
  - `setup-site.cmd` = `-Interactive` อยู่แล้ว (กด Enter ผ่าน default ได้); มือ: เติม `-Interactive` เอง
  - 🅿️ **port registry** `C:\gitops-sites.json` (server-local) จดจองให้อัตโนมัติ: denso = explicit ครั้งนี้ → ถูกบันทึก. **site ถัดไปไม่ต้องใส่ port เลย** → `setup-site.ps1 -Brand <ใหม่> -CreateDatabase` จองบล็อกถัดไป (9090.., redisDb ว่างถัดไป) เอง; พอร์ตชน brand อื่น → throw ก่อนสร้าง
  - ⚠️ **ใช้ `powershell.exe` (Windows PS 5.1) ไม่ใช่ `pwsh`** — IIS step ใช้ PSDrive `IIS:\` ซึ่ง pwsh7 โหลดผ่าน WinPSCompatSession แล้ว drive ไม่เกิด → `A drive with the name 'IIS' does not exist`
  - auto: `-Domain` (=`<brand>.local`), `-DatabaseName` (=`Vending_<Brand>`), `-SqlServerInstance` (detect MSSQLSERVER→`localhost` / Express→`localhost\SQLEXPRESS`) — ใส่ flag ทับได้ถ้าต่าง; repo public → ไม่ต้อง `-GitHubPat`
  - ⚠️ fail `Database creation failed` + `SSL Provider: certificate chain ... not trusted`? = setup-site เรียก sqlcmd ภายในขาด `-C` (ODBC18 trust self-signed cert). **pull gitops ล่าสุด** (fix แล้ว) หรือ workaround: สร้าง DB เอง แล้วรันซ้ำ **ตัด `-CreateDatabase`** (idempotent — ไม่ clone ใหม่/ไม่ทับ secret):
    ```powershell
    sqlcmd -S localhost -C -Q "IF DB_ID('denso_db') IS NULL CREATE DATABASE [denso_db];"
    ```
  - ⚠️ warning `.env.docker missing` = **harmless** — deploy อ่าน `.env.secrets` ก่อน (`.env.docker` = legacy fallback). ใส่ `DOCKER_USER/PASS` ใน `.env.secrets` พอ ไม่ต้องสร้าง `.env.docker`
- [ ] เติม `C:\gitops-denso\.env.secrets` — 🖱️ **ดับเบิลคลิก `C:\gitops-bootstrap\init-secrets.cmd`** → เลือก site → ถามทีละช่อง (มี help + SOURCE ในตัว) + ประกอบ DATABASE_URL ให้, `-Force` ทับ stub เก็บ `.bak`
  - **ที่มาแต่ละค่า** (auto = Enter ผ่าน; ที่เหลือกรอกเอง) — รายละเอียดเต็ม → [`tech-team-handbook.md` §1.3](tech-team-handbook.md):

    | Key | ที่มา |
    |---|---|
    | `BASE_URL` / `DATABASE_NAME` / `REDIS_DB` / `APM_SERVICE_NAME` / `JWT_*` | *auto* (registry / gen) — Enter ผ่าน |
    | `DB_HOST` (+`REDIS_HOST`/`SMTP_HOST`) | machine name ของ server (`hostname`) — **ห้าม localhost** |
    | `DB_PASSWORD` | sa password (ตั้งตอน Mixed Mode + sa, Phase 3) |
    | `REDIS_PASSWORD` | = ค่าที่ใส่ตอน setup Redis (default ตรงกัน) |
    | `MACHINE_CREATE_PASSWORD` | กำหนดเอง — ใช้ register ตู้ในแอป |
    | `VITE_API_KEY` | *auto-gen* — key ของ web เอง (web ใช้ JWT, API ไม่เช็ค x-api-key) → Enter ผ่าน. ไม่เกี่ยวกับ api-key ของตู้ |
    | `DOCKER_USER` / `DOCKER_PASS` | Docker Hub account + **access token** (hub.docker.com > PAT, Read-only) |
    | `ELASTIC_APM_SECRET_TOKEN` | token ฝั่ง ELK (`vending-machine-elk`) — **ว่าง** ถ้าไม่ ship เข้า ELK กลาง |
    | `TURNSTILE_*` / `LOCIZE_*` | Cloudflare / Locize dashboard — ว่าง/`0` ถ้าไม่ใช้ (offline) |
  - หรือแก้มือใน editor ก็ได้
  - pre-stage `-KeepSecrets`? → restore แทน (ข้าม init-secrets): `Copy-Item C:\gitops-secrets-backup\denso.env.secrets C:\gitops-denso\.env.secrets -Force`
- [ ] เช็ค placeholder ว่าง (ต้องไม่เจอ):
  ```powershell
  Select-String C:\gitops-denso\.env.secrets -Pattern 'your_.*_here|generate_a_64'
  ```
- [ ] start + deploy:
  ```powershell
  nssm start denso-gitops-deploy        # pull image (instant ถ้า warm) + migration + blue-green up
  ```
- [ ] 🔒 **เปิด HTTPS (จำเป็น!)** — 🖱️ `0-START-HERE.cmd` → เมนู **9** → option **2** (self-signed `denso.local`):
  ```powershell
  # หรือ: powershell -File C:\gitops-bootstrap\set-https.ps1 -Brand denso -SelfSigned
  ```
  - ⚠️ **ห้ามข้าม** — web.config redirect `http://` → `https://` เสมอ. ไม่เปิด HTTPS = เปิดเว็บแล้ว **ERR_CONNECTION_REFUSED** (redirect ไป 443 ที่ไม่มีใครฟัง)
  - ตู้/client: import `denso.local.cer` เข้า Trusted Root (set-https พิมพ์คำสั่ง export/import ให้ตอนจบ) ไม่งั้น browser เตือน
- [ ] verify + โชว์:
  ```powershell
  pwsh -File .\doctor.ps1 -Brand denso   # FAIL=0 (https binding ต้อง PASS)
  ```
  → เปิด portal **`https://denso.local`** → กดผ่าน warning self-signed → **admin login เข้า dashboard**
- [ ] 🔑 **login admin ตั้งต้น** (seed จาก migration — เหมือนกันทุก site):
  - user: `admin@ttfts.com`  ·  password: `1234`
  - ⚠️ **เปลี่ยน password ทันทีหลัง login** (default อ่อน + ใช้ร่วมทุก site = เสี่ยง) แล้วสร้าง admin ของ site/ลูกค้าเอง

## Phase 5 — ตู้ Android (รายละเอียด: §2)
- [ ] ลงแอป + set Device Owner → Settings พิมพ์มือ Server URL + API key (ไม่มี QR scanner) → sync → **dispense test 1 รายการ**

## Phase 6 — ส่งมอบ
- [ ] ตัดเน็ต → `deploy.ps1` idle (pull fail harmless, service รันต่อ) = ปกติ

---

**พังตรงไหน → [`tech-team-handbook.md` ภาคผนวก B](tech-team-handbook.md)** (502 / restart-loop / git-pull ค้าง / HNS hang / compose-hash ไม่เปลี่ยน)
