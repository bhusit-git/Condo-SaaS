# SkillToSus

SkillToSus คือโปรเจกต์ออกแบบระบบ Condo Management SaaS สำหรับบริหารงานนิติ/คอนโดแบบ multi-tenant โดยเชื่อมกับ LINE LIFF และ LINE Messaging API เพื่อให้ทีมแอดมิน พนักงาน และลูกบ้านทำงานร่วมกันผ่านเว็บและ LINE ได้ในระบบเดียว

เอกสารใน repo นี้เป็น product plan, implementation contract, domain glossary และ ADR สำหรับการพัฒนา v1 ของระบบ

## Project Summary

ระบบนี้ออกแบบมาเพื่อให้แพลตฟอร์มสามารถดูแลหลายองค์กรและหลายคอนโดได้ โดยแต่ละคอนโดมีข้อมูลอาคาร ชั้น ห้อง ลูกบ้าน พนักงาน สิทธิ์การใช้งาน และการตั้งค่า LINE ของตัวเอง

v1 โฟกัส flow หลักสำหรับ pilot:

1. สร้าง Organization และ Condo
2. นำเข้าข้อมูลอาคาร ห้อง และลูกบ้านผ่าน CSV
3. ให้ลูกบ้าน bind LINE ผ่าน Shared Platform LINE OA และ LIFF deep link/QR
4. ให้พนักงานรับพัสดุของลูกบ้าน
5. ส่ง LINE notification แจ้งลูกบ้าน
6. ให้ลูกบ้านเปิด LIFF เพื่อดูสถานะพัสดุและประกาศ

## Main Users

- **Platform/Admin**: สร้างองค์กรและคอนโด ตั้งค่าระบบ เริ่มต้น pilot
- **Condo Admin**: จัดการข้อมูลคอนโด พนักงาน สิทธิ์ และการตั้งค่า
- **Condo Manager / Juristic Staff**: จัดการงานนิติ เช่น ประกาศ ลูกบ้าน พัสดุ และ import
- **Security Staff**: รับพัสดุ ส่งมอบพัสดุ และส่งประกาศฉุกเฉินเมื่อได้รับสิทธิ์
- **Resident**: ผูกบัญชี LINE กับห้อง ดูประกาศ และดูสถานะพัสดุผ่าน LIFF

## Core Features

### Condo Setup

- สร้าง Organization และ Condo
- ตั้งค่า Building, Floor, Unit
- สร้าง staff user แรกของคอนโด
- seed preset roles และ permission defaults
- สร้าง LIFF deep link / QR สำหรับแต่ละคอนโด

### CSV Import

- รองรับ CSV UTF-8 สำหรับข้อมูล layout และ resident
- มี template แยกสำหรับ `unit_layout.csv` และ `residents.csv`
- ตรวจ diff ก่อน apply
- รองรับ scoped import เช่น ทั้งคอนโด เฉพาะอาคาร หรือบางชั้น
- missing rows เป็น warning/proposed change ไม่ลบข้อมูลอัตโนมัติ
- ตรวจ stale conflict เมื่อข้อมูลถูกแก้ในเว็บหลัง import รอบก่อน

### LINE Binding

- ใช้ Shared Platform LINE OA เป็น default ใน phase 1
- ลูกบ้านเข้า context คอนโดผ่าน LIFF deep link หรือ QR เฉพาะคอนโด
- auto-bind เมื่อ `unit + normalized phone` match กับ active resident ได้เพียงคนเดียว
- กรณี name-only, phone mismatch หรือ ambiguous จะเข้าสู่ staff review
- block duplicate LINE account ในคอนโดเดียวกัน
- รองรับอนาคตสำหรับ Custom LINE OA ใน phase 2

### Parcels

- พนักงานบันทึกพัสดุให้ห้อง
- รูปภาพเป็น optional
- ลูกบ้าน active ของห้องสามารถเห็นสถานะพัสดุได้
- การรับพัสดุคืนยืนยันโดย staff พร้อมชื่อผู้รับ note หรือรูป optional
- ส่ง LINE notification ตาม policy ของคอนโด เช่น owner/tenant/all/none

### Announcements

- ประกาศถึงทั้งคอนโด อาคาร ชั้น ห้อง ช่วงห้อง role ลูกบ้าน หรือ resident เฉพาะคน
- snapshot recipients ตอน publish ทุกครั้ง
- รองรับ publish only, publish + LINE notify และ critical notification
- v1 รองรับ text และรูปภาพ 1 รูป

### Notifications

- รองรับ priority: `critical`, `operational`, `broadcast`
- ใช้ reply message เมื่อมี active LINE interaction
- ใช้ individual push สำหรับ event เฉพาะกลุ่มหรือจำนวนน้อย
- ใช้ multicast สำหรับประกาศกลุ่มใหญ่
- มี quota guardrail ต่อคอนโดเมื่อใช้ Shared Platform LINE OA
- บันทึกสถานะ delivery แบบแยกความแน่นอน เช่น `sent_confirmed` และ `sent_assumed`
- ใช้ Postgres-backed queue และ scheduled Edge Function worker

## Architecture

Stack ที่กำหนดไว้สำหรับ v1:

- **Frontend**: React Vite แยก entrypoint เป็น `/admin`, `/staff`, `/liff`
- **Backend**: Supabase Postgres, Auth, Storage, Edge Functions
- **Resident App**: LINE LIFF
- **Messaging**: LINE Messaging API
- **Queue**: Postgres queue tables + scheduled Edge Function worker

แนวคิดสำคัญคือ route เป็นเพียง UX convention เท่านั้น ส่วน security boundary จริงอยู่ที่ Supabase RLS, Edge Functions/RPC และ permission checks ฝั่ง server

## v1 Scope

อยู่ใน v1:

- Onboarding คอนโดแบบ platform-controlled
- CSV import สำหรับ unit/resident data
- Staff/admin auth ด้วย username/password
- Resident LINE binding ผ่าน Shared Platform LINE OA
- Parcel receive/pickup workflow
- Announcements
- LINE notification queue และ delivery tracking
- Staff preset roles และ permission toggles
- Audit events สำหรับ action สำคัญ

ยังไม่อยู่ใน v1:

- Maintenance requests
- Billing
- Facility booking
- Visitor QR
- Documents
- Full incident case management
- Full custom role builder
- Staff self-service password reset
- Technician role
- Custom LINE OA admin/migration/re-bind UI แบบเต็ม

## Key Design Decisions

- `Organization` คือ customer/account และ `Condo` คือโครงการภายใต้องค์กร
- Resident เป็น identity ระดับ organization ไม่ duplicate ต่อห้อง
- Unit Resident คือความสัมพันธ์ระหว่าง resident กับห้อง เช่น owner, tenant, family
- v1 ใช้ Shared Platform LINE OA เพื่อให้ pilot เริ่มเร็ว
- Custom LINE OA เตรียม schema ไว้ แต่เป็น phase 2
- LIFF frontend ไม่ query customer-data table โดยตรง แต่เรียก Edge Functions/RPC ที่ verify LIFF identity ฝั่ง server
- Critical notification ต้องมี reason, scope confirmation, audit record และ rate/quota guardrail

## Documentation

- [Context Glossary](CONTEXT.md): glossary และ domain language ของระบบ
- [Condo SaaS + LINE LIFF Plan](docs/condo-saas-line-liff-plan-v6.1.md): product/architecture overview
- [v1 Implementation Contract](docs/v1-implementation-contract.md): source of truth สำหรับ data model, RLS, Edge Functions, storage, queue, import, audit และ tests
- [ADR 0001](docs/adr/0001-support-shared-and-custom-line-oa.md): shared/custom LINE OA strategy
- [ADR 0002](docs/adr/0002-use-line-multicast-for-large-announcements.md): multicast strategy สำหรับ announcement ขนาดใหญ่
- [ADR 0003](docs/adr/0003-resident-liff-access-through-edge-functions.md): LIFF access ผ่าน Edge Functions

## Repository Status

ตอนนี้ repo นี้เป็น documentation/specification repository สำหรับวางแผนและล็อก contract ของระบบก่อน implementation

