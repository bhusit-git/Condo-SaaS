# SkillToSus

SkillToSus คือโปรเจกต์ออกแบบระบบ Condo Management SaaS สำหรับบริหารงานนิติ/คอนโดแบบ multi-tenant โดยเริ่มจากระบบหลังบ้านหลักก่อน แล้วค่อยเพิ่ม LINE LIFF และ LINE Messaging API เป็น module ถัดไป เพื่อให้ทีมแอดมิน พนักงาน และลูกบ้านทำงานร่วมกันผ่านเว็บและ LINE ได้ในระบบเดียวเมื่อ phase พร้อม

เอกสารใน repo นี้เป็น product plan, implementation contract, domain glossary และ ADR สำหรับการพัฒนา v1.0 Core Backoffice และ roadmap phase ถัดไปของระบบ

## Project Summary

ระบบนี้ออกแบบมาเพื่อให้เป็นระบบเดียวที่รองรับเจ้าของหรือบริษัทที่ต้องดูแลหลายที่พัก หลายตึก หลายสาขา หรือหลายองค์กรได้ โดยแพลตฟอร์มสามารถดูแลหลายองค์กรและหลายคอนโด/ที่พักได้ ภายใต้แต่ละองค์กรสามารถมีหลายคอนโด/ที่พัก และแต่ละคอนโด/ที่พักมีข้อมูลอาคาร ชั้น ห้อง ลูกบ้าน พนักงาน สิทธิ์การใช้งาน และการตั้งค่า LINE ของตัวเอง

ในภาษาธุรกิจของหอพักหรือที่พักขนาดเล็ก คำว่า "สาขา" หรือ "ที่พัก" ให้ map กับ `Condo` ในโมเดล v1 ส่วน "ตึก" ให้ map กับ `Building` ภายในคอนโด/ที่พักนั้น เพื่อไม่ต้องสร้างระบบแยกหลายชุดเมื่อลูกค้ามีหลายสถานที่ต้องดูแล

v1.0 โฟกัส flow หลักสำหรับ Core Backoffice pilot:

1. สร้าง Organization และ Condo
2. นำเข้าข้อมูลอาคาร ห้อง และลูกบ้านผ่าน CSV
3. ตั้งค่า staff, preset roles, permissions และ package/subscription limits
4. ให้พนักงานรับ/ส่งมอบพัสดุของลูกบ้านผ่าน `/staff`
5. ให้แอดมินหรือนิติ publish ประกาศและดู recipient snapshot ในระบบ
6. ตรวจสอบ audit log, import history, parcel history, และ billing settings

## Main Users

- **Platform/Admin**: สร้างองค์กรและคอนโด ตั้งค่าระบบ เริ่มต้น pilot
- **Accommodation Owner / HQ**: เจ้าของกิจการหรือสำนักงานใหญ่ของลูกค้าที่ต้องดูภาพรวมหลายคอนโด/ที่พัก/สาขาภายใต้องค์กรเดียว ก่อนเลือกเข้าไปดูรายละเอียดของแต่ละที่
- **Condo Admin**: จัดการข้อมูลคอนโด พนักงาน สิทธิ์ และการตั้งค่า
- **Condo Manager / Juristic Staff**: จัดการงานนิติ เช่น ประกาศ ลูกบ้าน พัสดุ และ import
- **Security Staff**: รับพัสดุ ส่งมอบพัสดุ และส่งประกาศฉุกเฉินเมื่อได้รับสิทธิ์
- **Technician / Housekeeping Staff (v1.2)**: รับงานซ่อมหรืองานทำความสะอาดที่แอดมิน/นิติ assign ให้ กดเริ่มงาน และกดเสร็จพร้อมหลักฐานตาม policy ของคอนโด
- **Resident**: เป็นข้อมูลลูกบ้านใน v1.0; ใช้ LINE/LIFF เพื่อดูประกาศ พัสดุ และบริการลูกบ้านตั้งแต่ v1.1

## Core Features

### Condo Setup

- สร้าง Organization และ Condo
- กรอกข้อมูลคอนโดขั้นต่ำ: ชื่อคอนโด ที่อยู่ จังหวัด และรหัสไปรษณีย์
- ตั้งค่า Building, Floor, Unit ในหน้าผังห้อง
- สร้าง staff user แรกของคอนโด
- seed preset roles และ permission defaults
- ตั้งค่า package/subscription, usage limit, และ platform access status

### CSV Import

- รองรับ CSV UTF-8 สำหรับข้อมูล layout และ resident
- มี template แยกสำหรับ `unit_layout.csv` และ `residents.csv`
- ตรวจ diff ก่อน apply
- รองรับ scoped import เช่น ทั้งคอนโด เฉพาะอาคาร หรือบางชั้น
- missing rows เป็น warning/proposed change ไม่ลบข้อมูลอัตโนมัติ
- ตรวจ stale conflict เมื่อข้อมูลถูกแก้ในเว็บหลัง import รอบก่อน

### LINE + LIFF (v1.1)

- ใช้ Shared Platform LINE OA เป็น default ใน v1.1
- ลูกบ้านเข้า context คอนโดผ่าน LIFF deep link หรือ QR เฉพาะคอนโด
- auto-bind เมื่อ `unit + normalized phone` match กับ active resident ได้เพียงคนเดียว
- กรณี name-only, phone mismatch หรือ ambiguous จะเข้าสู่ staff review
- block duplicate LINE account ในคอนโดเดียวกัน
- รองรับอนาคตสำหรับ Custom LINE OA ใน Enterprise / phase ถัดไป

### Parcels

- พนักงานบันทึกพัสดุให้ห้อง
- รูปภาพเป็น optional
- v1.0 staff/admin เห็นสถานะและประวัติพัสดุในระบบหลังบ้าน
- ตั้งแต่ v1.1 ลูกบ้าน active ของห้องสามารถเห็นสถานะพัสดุผ่าน LIFF ได้
- การรับพัสดุคืนยืนยันโดย staff พร้อมชื่อผู้รับ note หรือรูป optional
- LINE notification ตาม policy ของคอนโดเริ่มใน v1.1

### Maintenance / Cleaning Requests (v1.2)

- ลูกบ้านหรือ staff สร้าง request งานซ่อมหรืองานทำความสะอาด
- แอดมิน/นิติเป็นคนตรวจและ assign งานให้ Technician หรือ Housekeeping Staff
- ผู้รับงานเห็นเฉพาะงานที่ได้รับมอบหมาย กดรับงาน เริ่มงาน และกดเสร็จผ่าน `/staff`
- Condo Admin ตั้งค่าได้ว่าตอนกดเสร็จงานต้องมีรูปหลักฐานหรือไม่ โดย default ให้เปิดไว้
- v1.2 ยังไม่เปิดคิวงานให้ช่างหรือแม่บ้านกดรับเองตามหมวดหมู่; เก็บไว้เป็น future enhancement

### Announcements

- ประกาศถึงทั้งคอนโด อาคาร ชั้น ห้อง ช่วงห้อง role ลูกบ้าน หรือ resident เฉพาะคน
- snapshot recipients ตอน publish ทุกครั้ง
- v1.0 รองรับ publish ภายในระบบและ recipient snapshot
- v1.1 เพิ่ม publish + LINE notify และ critical LINE notification
- v1.0 รองรับ text และรูปภาพ 1 รูป

### Notifications

- v1.0 บันทึก notification intent, audit, และ recipient snapshot ภายในระบบ
- v1.1 เพิ่ม LINE delivery queue, reply/push/multicast, quota guardrail,
  delivery status เช่น `sent_confirmed` และ `sent_assumed`
- LINE delivery ใช้ Postgres-backed queue และ scheduled Edge Function worker ตั้งแต่ v1.1

## Architecture

Stack ที่กำหนดไว้สำหรับ v1.0:

- **Frontend**: React Vite แยก entrypoint เป็น `/admin`, `/staff`
- **Backend**: Supabase Postgres, Auth, Storage, Edge Functions
- **Queue**: Postgres queue tables + scheduled Edge Function worker
- **Platform Owner**: `/admin/platform` สำหรับ Super Admin ภายใน admin app เดิม

เพิ่มใน v1.1:

- **Resident App**: LINE LIFF ที่ `/liff`
- **Messaging**: LINE Messaging API, Shared Platform LINE OA, LINE webhook

แนวคิดสำคัญคือ route เป็นเพียง UX convention เท่านั้น ส่วน security boundary จริงอยู่ที่ Supabase RLS, Edge Functions/RPC และ permission checks ฝั่ง server

## v1.0 Scope

อยู่ใน v1.0:

- Onboarding คอนโดแบบ platform-controlled
- CSV import สำหรับ unit/resident data
- Staff/admin auth ด้วย username/password
- Parcel receive/pickup workflow
- Announcements
- Staff preset roles และ permission toggles
- Super Admin control plane แบบ minimal สำหรับเจ้าของแพลตฟอร์ม:
  subscription state, suspend/reactivate tenant, delayed usage metrics
- Commercial package/plan/subscription limits สำหรับ Basic, Business, Enterprise
- Billing settings เฉพาะการตั้งค่า: ค่าเช่า ค่าน้ำ ค่าไฟ ค่าปรับล่าช้า
  และตัวอย่าง/preview สูตรคำนวณในหน้าตั้งค่า
- Audit events สำหรับ action สำคัญ

## Commercial Packages

แพ็กเกจราคาเป็น commercial contract ของแพลตฟอร์ม และต้องถูกกำหนดตั้งแต่ต้น
เพื่อไม่ให้ implementation ไปผูก feature flag, quota, permission, หรือ
subscription logic แบบแก้ทีหลังยาก

หลักการสำคัญ:

- Package entitlement คือสิทธิ์ทางการค้า ไม่ใช่การยืนยันว่า module นั้นอยู่ใน
  v1.0 runtime แล้ว
- ฟีเจอร์ future-phase ที่อยู่ในแพ็กเกจหมายถึงลูกค้าในแพ็กเกจนั้นมีสิทธิ์ใช้งานเมื่อ
  module นั้น build และเปิดใช้งานตาม phase แล้ว
- Resident billing จริงยังเริ่มที่ v1.3; ใน v1.0 มีเฉพาะ Billing settings เท่านั้น
- Basic เป็น operations-first plan ที่ไม่รวม LINE/LIFF notification entitlement
  โดย default ส่วน Business ขึ้นไปคือ plan หลักสำหรับ LINE-enabled module ตั้งแต่ v1.1

| Feature / Limit | Basic | Business | Enterprise |
| --- | --- | --- | --- |
| ราคาเริ่มต้น | 199 บาท/เดือน/สาขา | 599 บาท/เดือน/สาขา | ติดต่อทีม |
| สาขา / คอนโดรวม | 1 สาขา | 1 สาขา | ไม่จำกัด |
| ห้องต่อสาขา | 30 ห้อง | 100 ห้อง | ไม่จำกัด |
| Staff account | 5 คน | 20 คน | ไม่จำกัด |
| จัดการตึก / ชั้น / ห้อง | included | included | included |
| จัดการลูกบ้าน | included | included | included |
| นำเข้าข้อมูล CSV | included | included | included |
| รับ / ส่งมอบพัสดุ | included | included | included |
| ประกาศ | included | included | included |
| Preset roles และ permissions | included | included | included |
| Audit log | included | included | included |
| Billing settings (v1.0) | included | included | included |
| Resident Billing runtime entitlement (v1.3) | included when shipped | included when shipped | included when shipped |
| LINE Binding + LIFF app (v1.1) | not included | included when shipped | included when shipped |
| LINE แจ้งเตือนพัสดุ + ประกาศ (v1.1) | not included | included when shipped | included when shipped |
| LINE ประกาศฉุกเฉิน (v1.1) | not included | included when shipped | included when shipped |
| Quota LINE/เดือน | not included | 1,000 messages | ไม่จำกัดตาม commercial cap |
| Custom LINE OA | not included | not included | included |
| รายรับ / รายจ่าย | not included | included when shipped | included when shipped |
| สต็อควัสดุ | not included | included when shipped | included when shipped |
| เบิกของช่าง / Work Order | not included | included when shipped | included when shipped |
| Maintenance Request | not included | included from v1.2 | included from v1.2 |
| Facility Booking | not included | included when shipped | included when shipped |
| Visitor QR | not included | included when shipped | included when shipped |
| Custom Role Builder | not included | not included | included when shipped |
| Documents module | not included | not included | included when shipped |
| Organization HQ dashboard | not included | not included | included when shipped |
| Email support | included | included | included |
| Priority support | not included | included | included |
| SLA + Onboarding support | not included | not included | included |

Add-on:

- Basic เพิ่มสาขา / คอนโด: 199 บาท/เดือน/สาขา แต่ละสาขาได้สิทธิ์ Basic 30 ห้อง
- Basic ห้องเกิน 30 ห้อง: 6 บาท/ห้อง/เดือน นับเฉพาะส่วนเกินต่อสาขา
- Business เพิ่มสาขา / คอนโด: 500 บาท/เดือน/สาขา แต่ละสาขาได้สิทธิ์ Business
  100 ห้อง + LINE entitlement
- Business ห้องเกิน 100 ห้อง: 6 บาท/ห้อง/เดือน นับเฉพาะส่วนเกินต่อสาขา
- Enterprise ไม่มี add-on มาตรฐาน ราคา custom รวม quota/สาขา/ห้องตามสัญญา

ยังไม่อยู่ใน v1:

- LINE/LIFF, LINE Binding, LINE webhook, LINE notification queue และ LINE delivery
  อยู่ใน v1.1 ไม่ใช่ v1.0 Core Backoffice
- Maintenance / Cleaning requests with Technician and Housekeeping Staff roles
  อยู่ใน v1.2 ไม่ใช่ v1.0 Core Backoffice
- Resident billing runtime, rent charge, water/electricity bills, invoice
  lifecycle, meter reading persistence, persisted charges, payment collection,
  accounting rules, และ payment gateway อยู่ใน v1.3
- Billing / Utility Bills LINE notifications สำหรับค่าเช่า ค่าน้ำ ค่าไฟ
- Facility booking
- Visitor QR
- Documents
- Full incident case management
- Full custom role builder
- Staff self-service password reset
- Custom LINE OA admin/migration/re-bind UI แบบเต็ม

## Key Design Decisions

- `Organization` คือ customer/account และ `Condo` คือโครงการภายใต้องค์กร
- เจ้าของหรือบริษัทหนึ่งรายสามารถมีหลาย `Condo` ได้; สำหรับตลาดหอพักหรือที่พักขนาดเล็ก `Condo` ใช้แทนสาขา/ที่พักหนึ่งแห่ง และ `Building` ใช้แทนตึกภายในสาขา/ที่พักนั้น
- Owner/HQ ของลูกค้าควรเห็นภาพรวมระดับองค์กรก่อน แล้วค่อยเลือกเข้าไปจัดการคอนโด/ที่พัก/สาขาที่ต้องการ; สิทธิ์พนักงานยังต้องจำกัดตาม scope ที่ได้รับ เช่น เฉพาะบางคอนโด/ที่พัก บางตึก หรือบาง workflow
- Resident เป็น identity ระดับ organization ไม่ duplicate ต่อห้อง
- Unit Resident คือความสัมพันธ์ระหว่าง resident กับห้อง เช่น owner, tenant, family
- v1.0 ไม่ต้องพึ่ง LINE/LIFF เพื่อให้ Basic และระบบหลังบ้านเริ่มใช้งานได้จริง
- v1.1 ใช้ Shared Platform LINE OA สำหรับ Business/Enterprise ที่เปิด LINE entitlement
- Custom LINE OA เตรียม schema ไว้ แต่เป็น Enterprise / phase ถัดไป
- Super Admin ใช้ Supabase Auth custom claim
  `app_metadata.role = "platform_super_admin"` และอยู่ที่ `/admin/platform`
- Subscription status ของเจ้าของ SaaS แยกจาก resident billing; runtime
  suspend ใช้ flag บน Organization/Condo ไม่ join subscription history ทุก write
- Usage metrics เป็น hourly/daily aggregate ไม่ใช่ realtime trigger counter
- เมื่อ v1.1 เปิด LIFF แล้ว LIFF frontend ไม่ query customer-data table โดยตรง แต่เรียก Edge Functions/RPC ที่ verify LIFF identity ฝั่ง server
- Critical notification ต้องมี reason, scope confirmation, audit record และ rate/quota guardrail
- Billing settings ใน v1 เป็น configuration/metadata เท่านั้น; ยังไม่สร้าง
  invoice, ไม่บันทึกยอดจริง, ไม่รับชำระเงิน และไม่ส่ง billing notification

## Roadmap

### LINE + LIFF v1.1

- เพิ่ม Shared Platform LINE OA สำหรับ Business/Enterprise
- เพิ่ม `/liff`, LIFF deep link / QR, LINE Binding, LINE webhook และ LINE notification queue
- ส่ง LINE notification สำหรับพัสดุ ประกาศ และ critical announcements
- ใช้ quota guardrail ต่อคอนโดและ delivery status ที่แยกความแน่นอน
- Basic ยังไม่รวม LINE/LIFF โดย default ยกเว้นมี paid upgrade, trial override หรือ entitlement เฉพาะ

### Maintenance / Cleaning v1.2

- เพิ่ม preset roles `Technician` และ `Housekeeping Staff`
- แอดมิน/นิติ assign งานให้ผู้รับผิดชอบก่อนในรอบแรก
- ผู้รับงานกดรับงาน เริ่มงาน และ resolve งานผ่าน `/staff`
- Condo Admin เปิด/ปิด policy รูปหลักฐานตอนปิดงานได้ต่อคอนโด
- อนาคตค่อยเพิ่มหมวดหมู่/skill matching ให้ช่างหรือแม่บ้านเห็นงานที่ตรงประเภทและกดรับเองได้

### Resident Billing v1.3

- เพิ่มระบบออกบิลจริงหลัง v1.2 Maintenance/Cleaning โดยยังคง v1.0 เดิมเป็น
  billing settings เท่านั้น
- บิลต้องผูกกับการอยู่อาศัยจริงผ่าน `lease_agreement_id` และ
  `tenant_unit_resident_id` ไม่ใช่แค่ `unit_id` หรือ `resident_id`
- รองรับ billing cycle, meter reading, invoice, invoice line items, manual
  payment record, และสถานะบิล `draft`, `issued`, `partially_paid`, `paid`,
  `overdue`, `void`
- เลขบิลใช้ `invoice_no` unique ต่อ Condo ด้วยรูปแบบ v1.3
  `INV-{YYYYMM}-{running_no}` เช่น `INV-202606-0001`; `YYYYMM` มาจาก billing
  cycle ไม่ใช่วันที่กด issue
- ถ้าห้องเปลี่ยนผู้เช่ากลางรอบบิล ต้องแยก invoice ตาม lease context และคิด
  prorate ค่าเช่าตามจำนวนวันที่ lease overlap กับรอบบิล
- LINE/LIFF ใช้เพื่อดูบิลและแจ้งเตือนเฉพาะผู้เช่าที่ยังมี active allowed
  context; ผู้เช่าที่ย้ายออกและถูก revoke LINE Binding แล้วไม่เห็นบิลผ่าน
  LIFF
- v1.3 ยังไม่รวม payment gateway, PDF invoice, tax invoice, accounting export,
  deposit settlement, post-move-out resident portal, หรือ bank reconciliation

### Multi-Site Owner / HQ

- Dashboard รวมสำหรับเจ้าของกิจการหรือสำนักงานใหญ่ที่มีหลายคอนโด/ที่พัก/สาขา
- เลือกเข้าไปดูรายละเอียดของแต่ละคอนโด/ที่พัก/สาขาหลังจากเห็นภาพรวม
- จัดการสิทธิ์พนักงานตามขอบเขตที่รับผิดชอบ เช่น ทั้งองค์กร เฉพาะคอนโด/ที่พัก เฉพาะตึก หรือเฉพาะ workflow
- คำว่า Owner/HQ ในส่วนนี้หมายถึงเจ้าของกิจการของลูกค้า ไม่ใช่ Platform Super Admin เจ้าของ SaaS และไม่ใช่ resident role `owner`

## Documentation

- [Context Glossary](CONTEXT.md): glossary และ domain language ของระบบ
- [Condo SaaS + LINE LIFF Plan](docs/condo-saas-line-liff-plan-v6.1.md): product/architecture overview และ phase roadmap
- [v1 Implementation Contract](docs/v1-implementation-contract.md): source of truth สำหรับ data model, RLS, Edge Functions, storage, queue, import, audit และ tests
- [Deployment Flow](docs/deployment-flow.md): Cloudflare Pages, Supabase, environment promotion, verification และ rollback contract สำหรับ v1.0; LINE/LIFF เพิ่มใน v1.1
- [ADR 0001](docs/adr/0001-support-shared-and-custom-line-oa.md): shared/custom LINE OA strategy
- [ADR 0002](docs/adr/0002-use-line-multicast-for-large-announcements.md): multicast strategy สำหรับ announcement ขนาดใหญ่
- [ADR 0003](docs/adr/0003-resident-liff-access-through-edge-functions.md): LIFF access ผ่าน Edge Functions

## Repository Status

ตอนนี้ repo นี้เป็น documentation/specification repository สำหรับวางแผนและล็อก contract ของระบบก่อน implementation
