# Buri Design System

เอกสารนี้สรุปทิศทางแบรนด์ ระบบสี typography component guidelines และ design tokens สำหรับ **Buri — จัดการที่พักในไม่กี่คลิก** โดยอิงจากโลโก้ที่ให้มาเป็นหลัก: โทนฟ้าสดใส น่าเชื่อถือ ใช้ง่าย เป็นมิตร และมีคาแรกเตอร์ผู้ช่วยที่สื่อถึงการจัดการที่พักแบบอัตโนมัติ

## 1. Brand Direction

**Buri** คือระบบจัดการที่พักสำหรับหอพัก และที่พักขนาดเล็ก ที่ช่วยให้เจ้าของและพนักงานจัดการทุกอย่างได้ง่ายขึ้นในไม่กี่คลิก

### Brand Personality

| บุคลิกแบรนด์ | ความรู้สึก |
| --- | --- |
| Friendly | ใช้งานง่าย ไม่ซับซ้อน |
| Smart | ช่วยจัดการงานแทนเจ้าของ |
| Trustworthy | ข้อมูลชัดเจน ปลอดภัย เชื่อถือได้ |
| Clean | หน้าจอสะอาด อ่านง่าย |
| Fast | ทำงานได้เร็ว ลดขั้นตอน |

### Core Brand Message

> Buri ช่วยให้การจัดการที่พักง่ายขึ้น ตั้งแต่ห้องพัก การจอง ลูกค้า การชำระเงิน ไปจนถึงรายงาน ทั้งหมดจบได้ในไม่กี่คลิก

### Brand Keywords

- ฟ้าใส
- สะอาด
- เป็นมิตร
- คล่องตัว
- ธุรกิจที่เข้าถึงง่าย
- ผู้ช่วยอัตโนมัติ

## 2. Visual Style

แนวทางภาพรวมของ Buri ควรอิงจากภาษาภาพของโลโก้:

- ใช้เส้นโค้งและมุมมนเป็นหลัก
- ใช้สีฟ้าเป็นแกนหลักของแบรนด์
- ใช้ gradient จากฟ้าอ่อนสู่ฟ้าเข้มในจุดสำคัญ
- ใช้สีเขียว lime เป็นสีของ success, confirm, paid state
- ใช้ mascot หรือ illustration ที่นุ่ม น่ารัก แต่ยังดูเป็นระบบธุรกิจ
- หลีกเลี่ยงสีดำแข็งจัด มุมแหลม หรือ layout ที่ดูเป็นระบบหลังบ้านแบบเก่า

### Mood & Tone

> Clean SaaS + Friendly Mascot + Soft Hotel Management

## 3. Color System

### Primary Colors

| Token | สี | Hex | การใช้งาน |
| --- | --- | --- | --- |
| Primary Blue 600 | ฟ้าหลักเข้ม | `#3776DA` | ปุ่มหลัก, active menu, header |
| Primary Blue 500 | ฟ้าหลัก | `#5598EE` | CTA, icon, link |
| Primary Blue 400 | ฟ้าสด | `#6AACF7` | hover, chart |
| Primary Blue 300 | ฟ้าอ่อน | `#8DC4FA` | card highlight |
| Primary Blue 100 | ฟ้าพื้นหลัง | `#EAF5FF` | section background |

### Secondary / Accent

| Token | สี | Hex | การใช้งาน |
| --- | --- | --- | --- |
| Lime 500 | เขียวสำเร็จ | `#ADDB7B` | success, check, paid |
| Lime 400 | เขียวสด | `#D1E76D` | badge, highlight |
| Navy 900 | น้ำเงินเข้ม | `#071E42` | text หลัก, icon เข้ม |
| Soft Sky | ฟ้าขาว | `#EDF4FC` | background หลัก |
| White | ขาว | `#FFFFFF` | card, modal, content area |

### Semantic Colors

| สถานะ | สี | Hex | การใช้งาน |
| --- | --- | --- | --- |
| Success | เขียว | `#8DD85F` | ชำระแล้ว, เช็กอินสำเร็จ |
| Warning | เหลืองส้ม | `#F6B84B` | รอชำระ, ใกล้ครบกำหนด |
| Danger | แดงนุ่ม | `#EF5B5B` | ยกเลิก, ค้างชำระ, error |
| Info | ฟ้า | `#5598EE` | แจ้งเตือนทั่วไป |
| Neutral | เทาอมฟ้า | `#EEF3F8` | disabled, border, empty state |

## 4. Gradient

Gradient เป็นลายเซ็นสำคัญของ Buri โดยใช้กับปุ่มหลัก การ์ดหัวข้อ hero card และภาพประกอบบางส่วน

```css
--gradient-primary: linear-gradient(135deg, #8DC4FA 0%, #5598EE 45%, #3776DA 100%);
--gradient-soft: linear-gradient(135deg, #EDF4FC 0%, #EAF5FF 100%);
--gradient-success: linear-gradient(135deg, #D1E76D 0%, #ADDB7B 100%);
```

### Recommended Usage

- ปุ่ม `เพิ่มการจอง`
- การ์ด `รายได้วันนี้`
- header ของ dashboard
- badge ยืนยันสำเร็จ
- empty state illustration

## 5. Typography

Typography ต้องอ่านภาษาไทยง่าย ดูร่วมสมัย และไม่แข็งเกินไป

### Recommended Fonts

| ประเภท | Font |
| --- | --- |
| Thai UI | `LINE Seed Sans TH` |
| Alternative | `Noto Sans Thai` |
| English / Number | `Inter` |

### Type Scale

| Style | ขนาด | น้ำหนัก | การใช้งาน |
| --- | --- | --- | --- |
| Display | 36px | 700 | หน้า landing / headline ใหญ่ |
| H1 | 28px | 700 | หัวข้อหน้า |
| H2 | 24px | 700 | หัวข้อ section |
| H3 | 20px | 600 | หัวข้อ card |
| Body Large | 16px | 400-500 | ข้อความหลัก |
| Body | 14px | 400 | ตาราง, form |
| Caption | 12px | 400 | คำอธิบาย, helper text |
| Button | 15px | 600 | ปุ่ม |

## 6. Border Radius & Shape

เพราะโลโก้มีความโค้งมนสูง UI ของ Buri จึงควรใช้ภาษารูปทรงแบบ soft rounded ทั้งระบบ

| Element | Radius |
| --- | ---: |
| App Container | 28px |
| Card | 24px |
| Button | 16px |
| Input | 14px |
| Badge | 999px |
| Modal | 28px |
| Icon Container | 18px |

### Shape Guidance

- ใช้ปุ่มมุมมนและ card ขอบโค้งเสมอ
- ลด table แข็ง ๆ ด้วย container โค้งมนและระยะหายใจมากขึ้น
- หลีกเลี่ยง border เข้มและเส้นตรงหนาเกินไป

## 7. Spacing System

ใช้ระบบ 8px grid เพื่อให้ทั้ง dashboard, form และ card layout ดูเป็นระเบียบและสบายตา

```css
--space-1: 4px;
--space-2: 8px;
--space-3: 12px;
--space-4: 16px;
--space-5: 24px;
--space-6: 32px;
--space-7: 40px;
--space-8: 48px;
```

หลักสำคัญคือ layout ต้องโปร่งและอ่านง่าย เพื่อสื่อความรู้สึกว่า Buri ช่วยลดความวุ่นวาย ไม่ได้เพิ่มความซับซ้อน

## 8. Component System

### 8.1 Buttons

#### Primary Button

ใช้กับ action สำคัญ เช่น เพิ่มการจอง บันทึก หรือยืนยันการชำระเงิน

```css
background: linear-gradient(135deg, #5598EE, #3776DA);
color: #FFFFFF;
border-radius: 16px;
height: 48px;
font-weight: 600;
box-shadow: 0 12px 24px rgba(55, 118, 218, 0.24);
```

ตัวอย่างข้อความ:

- เพิ่มการจอง
- เปิดห้องพัก
- บันทึกข้อมูล
- ยืนยันการชำระเงิน

#### Secondary Button

ใช้กับ action รองหรือ secondary workflow

```css
background: #EAF5FF;
color: #3776DA;
border: 1px solid #CFE6FF;
```

#### Success Button

ใช้กับ action confirm

```css
background: linear-gradient(135deg, #D1E76D, #ADDB7B);
color: #071E42;
```

#### Danger Button

ใช้เฉพาะ action ที่มีความเสี่ยง เช่น ลบ ยกเลิก หรือ revert

```css
background: #FFF0F0;
color: #EF5B5B;
```

### 8.2 Cards

Card ของ Buri ควรเป็นสีขาว ขอบฟ้าอ่อน เงานุ่ม และดูเบาสบาย

```css
background: #FFFFFF;
border: 1px solid #E1EEFB;
border-radius: 24px;
box-shadow: 0 16px 40px rgba(7, 30, 66, 0.08);
padding: 24px;
```

#### Card Types

| Card | ใช้กับ |
| --- | --- |
| KPI Card | จำนวนห้องว่าง, รายได้, การจองวันนี้ |
| Room Card | สถานะห้องพักแต่ละห้อง |
| Booking Card | รายการจอง |
| Guest Card | ข้อมูลลูกค้า |
| Alert Card | แจ้งเตือนค้างชำระ / เช็กเอาต์วันนี้ |

### 8.3 Input Field

Input ต้องให้ความรู้สึกนุ่ม ใช้ง่าย และรองรับฟอร์มจำนวนมากโดยไม่กดดันผู้ใช้

```css
height: 48px;
border-radius: 14px;
border: 1px solid #D8E8FA;
background: #FFFFFF;
padding: 0 16px;
```

Focus state:

```css
border-color: #5598EE;
box-shadow: 0 0 0 4px rgba(85, 152, 238, 0.16);
```

### 8.4 Status Badge

ใช้สำหรับสถานะห้องพัก การจอง และสถานะการชำระเงิน

| สถานะ | สีพื้น | สีข้อความ |
| --- | --- | --- |
| ว่าง | `#EAF5FF` | `#3776DA` |
| เข้าพักอยู่ | `#EFFFF0` | `#4A9F35` |
| จองแล้ว | `#F4EDFF` | `#7C4DFF` |
| รอชำระ | `#FFF6E5` | `#B87500` |
| ชำระแล้ว | `#EFFFF0` | `#4A9F35` |
| ค้างชำระ | `#FFF0F0` | `#D94141` |
| ทำความสะอาด | `#EEF3F8` | `#52677A` |
| ซ่อมบำรุง | `#FFF4E8` | `#C46B1C` |

Badge ควรใช้ทรง pill:

```css
border-radius: 999px;
padding: 6px 12px;
font-size: 12px;
font-weight: 600;
```

## 9. Icon & Illustration Style

### Icon Direction

- ใช้เส้นมน
- น้ำหนักเส้น 2px
- ไม่มีมุมแหลม
- ใช้สี primary blue เป็นหลัก
- วางใน background circle สีฟ้าอ่อนเมื่อใช้กับเมนูหรือ quick action

### Suggested Icon Set

| หมวด | Icon |
| --- | --- |
| Dashboard | grid / chart |
| ห้องพัก | building / door |
| การจอง | calendar check |
| ลูกค้า | user |
| การเงิน | wallet |
| รายงาน | bar chart |
| ตั้งค่า | gear |
| แจ้งเตือน | bell |

### Mascot Usage

mascot จากโลโก้สามารถใช้เป็น “ผู้ช่วย Buri” ในจุดที่ต้องการความเป็นมิตรและการช่วยนำทางผู้ใช้

กรณีใช้งานที่เหมาะ:

- หน้า empty state
- หน้า onboarding
- popup แนะนำการใช้งาน
- success confirmation
- friendly error message

ตัวอย่างข้อความ:

> Buri ยังไม่พบการจองวันนี้ ลองเพิ่มการจองแรกของคุณได้เลย

## 10. Layout System

### Desktop Layout

เหมาะกับเจ้าของที่พัก ผู้ดูแล หรือ admin ที่ต้องจัดการหลายข้อมูลพร้อมกัน

```text
Sidebar | Main Content
```

Sidebar:

- โลโก้ Buri ด้านบน
- เมนูหลัก
- โปรไฟล์และการตั้งค่าด้านล่าง

Main Content:

- page header
- KPI cards
- quick actions
- table, calendar หรือ room grid ตามหน้าใช้งาน

### Mobile Layout

ใช้ bottom navigation เพื่อให้ใช้งานสะดวกบนมือถือ

เมนูหลักแนะนำ:

1. หน้าแรก
2. ห้องพัก
3. การจอง
4. ลูกค้า
5. เพิ่มเติม

ปุ่ม action หลักควรเป็น floating action button เช่น `+ เพิ่มการจอง`

## 11. Page Design Direction

### Dashboard

ต้องแสดงข้อมูลที่เจ้าของที่พักอยากเห็นทันทีเมื่อเปิดระบบ

#### KPI Cards

- ห้องว่าง
- ห้องที่มีผู้เข้าพัก
- รายได้วันนี้
- รอชำระเงิน
- เช็กอินวันนี้
- เช็กเอาต์วันนี้

#### Quick Actions

- เพิ่มการจอง
- เปิดห้องพักใหม่
- บันทึกรับเงิน
- แจ้งซ่อม
- เพิ่มลูกค้า

### Room Management Page

แสดงห้องพักในรูปแบบ card grid เพื่อให้สแกนสถานะได้เร็ว

แต่ละห้องควรมี:

- เลขห้อง
- ประเภทห้อง
- ราคา
- สถานะ
- ลูกค้าที่พักอยู่
- วันเช็กอิน / เช็กเอาต์
- ปุ่มจัดการ

ตัวอย่างการ์ด:

```text
ห้อง 204
Deluxe Room
฿1,200 / คืน

[เข้าพักอยู่]

ลูกค้า: คุณมิน
เช็กเอาต์: 28 มิ.ย. 2026
```

### Booking Page

ควรใช้มุมมองแบบ list + calendar เพื่อช่วยทั้งการจัดการรายวันและการวางแผนล่วงหน้า

ข้อมูลที่ควรมี:

- ชื่อลูกค้า
- ห้อง
- วันเข้าพัก
- วันออก
- สถานะการชำระเงิน
- ช่องทางจอง
- ยอดเงิน
- ปุ่มเช็กอิน / เช็กเอาต์

### Payment Page

เป็นหน้าที่ต้องอ่านชัดมาก เพราะเกี่ยวข้องกับเงินและสถานะการชำระ

ข้อมูลหลัก:

- ยอดที่ต้องชำระ
- ชำระแล้ว
- ค้างชำระ
- วิธีชำระเงิน
- ใบเสร็จ
- ประวัติการจ่าย

หลักการใช้สี:

- ใช้สีเขียวเมื่อจ่ายครบ
- ใช้สีส้มเมื่อรอชำระ
- ใช้สีแดงนุ่มเมื่อค้างชำระหรือมีปัญหา

### Customer Page

ข้อมูลลูกค้าไม่ควรแน่นเกินไป และควรอ่านเป็น block ที่เข้าใจง่าย

ควรมี:

- ชื่อ
- เบอร์โทร
- LINE / Email
- ประวัติการเข้าพัก
- เอกสารแนบ
- หมายเหตุ
- ระดับความน่าเชื่อถือ

## 12. Voice & Copywriting

Buri ควรพูดง่าย เป็นมิตร สุภาพ และไม่เป็นทางการจนห่างเหิน

### Button Copy Examples

| ไม่ควรใช้ | ควรใช้ |
| --- | --- |
| Submit | บันทึกข้อมูล |
| Confirm Transaction | ยืนยันการชำระเงิน |
| Create Reservation | เพิ่มการจอง |
| Manage Accommodation | จัดการที่พัก |
| View Details | ดูรายละเอียด |

### Empty State

```text
ยังไม่มีการจองวันนี้
เริ่มเพิ่มการจองแรก เพื่อให้ Buri ช่วยจัดการต่อได้เลย
```

### Success Message

```text
บันทึกสำเร็จแล้ว
Buri อัปเดตข้อมูลให้เรียบร้อย
```

### Error Message

```text
ยังบันทึกไม่ได้
กรุณาตรวจสอบข้อมูลที่จำเป็นอีกครั้ง
```

## 13. CSS Design Tokens

```css
:root {
  --color-primary-600: #3776DA;
  --color-primary-500: #5598EE;
  --color-primary-400: #6AACF7;
  --color-primary-300: #8DC4FA;
  --color-primary-100: #EAF5FF;

  --color-lime-500: #ADDB7B;
  --color-lime-400: #D1E76D;

  --color-navy-900: #071E42;
  --color-text-main: #071E42;
  --color-text-muted: #6B7A90;

  --color-bg-main: #EDF4FC;
  --color-bg-card: #FFFFFF;
  --color-border: #D8E8FA;

  --radius-xs: 8px;
  --radius-sm: 12px;
  --radius-md: 16px;
  --radius-lg: 24px;
  --radius-xl: 28px;
  --radius-pill: 999px;

  --shadow-card: 0 16px 40px rgba(7, 30, 66, 0.08);
  --shadow-button: 0 12px 24px rgba(55, 118, 218, 0.24);

  --gradient-primary: linear-gradient(135deg, #8DC4FA 0%, #5598EE 45%, #3776DA 100%);
  --gradient-success: linear-gradient(135deg, #D1E76D 0%, #ADDB7B 100%);
}
```

## 14. Design Summary

**Buri ควรเป็นระบบที่ดูน่าใช้มากกว่าระบบหลังบ้านทั่วไป** ไม่ควรดูแข็งหรือซับซ้อนเกินไป แต่ยังต้องน่าเชื่อถือพอสำหรับข้อมูลห้องพัก ลูกค้า การจอง และการเงิน

คำจำกัดความของดีไซน์ Buri คือ:

> ฟ้าใส สะอาด ใช้ง่าย น่าเชื่อถือ มีผู้ช่วยตัวน้อยคอยจัดการที่พักให้ทุกอย่างจบในไม่กี่คลิก
