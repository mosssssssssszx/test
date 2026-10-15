# การตรวจสูตร

ลูกค้าส่งข้อมูลเข้าสองช่องทาง ช่องหนึ่งตอบเป็นข้อความทันที อีกช่องหนึ่งตอบด้วยการย้ายไฟล์ หลังจากนั้นระบบเอาสองชุดมาเทียบกัน ผลเทียบถูกเก็บไว้ในระบบ ไม่ได้ส่งกลับไปให้ลูกค้า

รายชื่อโรงงานที่อนุญาตให้ตรวจชื่อ `BATCH_ALLOWED_PLANTS` ตอนนี้มีค่า `HHN`

---

## ลูกค้าส่งอะไร และได้อะไรกลับ

### ส่งเข้า `POST /api/auth/token`

ส่งรหัสลูกค้าเข้ามาเพื่อขอสิทธิ์ส่งสูตร

ถ้าถูกต้อง ลูกค้าได้

```json
{
  "access_token": "โทเคนสำหรับแนบไปกับคำขอถัดไป",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

`expires_in` 3600 คือใช้ได้ 1 ชั่วโมง ถ้ารหัสไม่ถูก จะได้ HTTP 401

### ส่งเข้า `POST /api/prodprc`

แนบ `access_token` แล้วส่งสูตรเข้ามา ชื่อฟิลด์ต้องตรงตัวพิมพ์ `Plant` ไม่เท่ากับ `plant` และ `param01` ไม่เท่ากับ `Param01`

ข้อมูลหนึ่งชุดมีหัวสูตรกับรายการขั้นตอน หัวสูตรใช้คำเหล่านี้

| คำ | ความหมาย |
|---|---|
| `Plant` | โรงงาน |
| `BulkCode` | รหัสสินค้า |
| `BulkName` | ชื่อสินค้า |
| `ProcessType` | รหัสกระบวนการ |
| `Revision` | ฉบับแก้ไข |
| `BatchSizeMin` | ล็อตต่ำสุดที่รับได้ |
| `BatchSizeMax` | ล็อตสูงสุดที่รับได้ |
| `Unit` | หน่วยของล็อต |

แต่ละขั้นตอนใช้คำเหล่านี้

| คำ | ความหมาย |
|---|---|
| `param01` | ถ้าเป็น `RM Code` ขั้นนี้คือขั้นใส่วัตถุดิบ |
| `param02` | รหัสวัตถุดิบที่ใช้ได้ หลายรหัสคั่นด้วย `\|` แปลว่าเลือกได้ตัวใดตัวหนึ่ง |
| `param03` | ถ้าเป็น `%RM` คู่กับ `param01` จึงนับเป็นขั้นวัตถุดิบ |
| `param04` | เปอร์เซ็นต์ของล็อต ถ้ามีตัวเลข ขั้นนี้ชื่อ `FIXED_PERCENT` |
| `Phase` | ชื่อขั้น ถ้าว่าง `param04` และมีคำว่า `ADJUST` ขั้นนี้ชื่อ `ADJUST` |

`FIXED_PERCENT` ล็อกทั้งรหัสและปริมาณ `ADJUST` ตรวจแค่รหัส ไม่ล็อกปริมาณ

ถ้ารับไว้ได้ ลูกค้าได้ HTTP 200

```json
{
  "status": "success",
  "message": "Production Process received successfully",
  "headerCount": 1,
  "stepCount": 36,
  "processStatus": "PENDING"
}
```

`PENDING` แปลว่ารับสูตรไว้แล้ว ยังไม่ได้แปลว่าใบสั่งผ่าน `headerCount` คือจำนวนสูตรในคำขอนั้น `stepCount` คือจำนวนขั้นตอนที่เก็บได้

คำตอบอื่น

| เหตุ | HTTP | ข้อความ |
|---|---|---|
| ไม่มีก้อน `Result` | 400 | Invalid JSON format: Result[] is required |
| ไม่มีโทเคน โทเคนผิด หรือโทเคนหมดอายุ | 401 | ข้อความเรื่องโทเคน |
| โทเคนไม่มีสิทธิ์ส่งสูตร | 403 | Insufficient scope |
| ส่งสูตรซ้ำ ทั้งที่ระบบตั้งห้ามซ้ำ | 409 | Duplicate Production Process พร้อม `processId`, `plant`, `bulkCode`, `processType`, `revision` |
| บันทึกไม่สำเร็จ | 500 | Save Production Process failed |

ตอนนี้ระบบตั้งให้รับสูตรซ้ำได้ ลูกค้าจึงได้ 200 แม้ส่งชุดเดิมเข้ามาอีกครั้ง

### ส่งไฟล์ `REPID_*.csv` และ `MASID_*.csv`

ลูกค้าวางไฟล์เข้าโฟลเดอร์รับไฟล์ ไม่มีข้อความ JSON กลับ

ไฟล์ `REPID` คือใบสั่งผลิต หนึ่งบรรทัดคือวัตถุดิบหนึ่งรายการ ค่าหัวใบสั่งซ้ำอยู่ในทุกบรรทัด คำที่ใช้ตรวจคือ

| คำในไฟล์ | ความหมาย |
|---|---|
| `MaterialCode` | รหัสสินค้าที่จะผลิต |
| `ProcessNo` | รหัสกระบวนการ ระบบใช้แถวแรกที่มีค่า |
| `TargetQty` | ขนาดล็อต |
| `TargetUnit` | หน่วยของล็อต เช่น `KG` |
| `RequirementQty` | ปริมาณของรหัสวัตถุดิบนั้น |
| `Unit` | หน่วยของวัตถุดิบนั้น เช่น `KG` หรือ `G` |

รหัสโรงงานในไฟล์ `REPID` เช่น `30` ไม่ถูกใช้ตอนตรวจ โรงงานที่เอามาตัดสินคือ `Plant` ที่ส่งเข้า `POST /api/prodprc`

ไฟล์ `MASID` คือรายชื่อวัสดุ ไม่เข้าการเทียบสูตร

ระบบจะอ่านไฟล์เมื่อเห็นขนาดไฟล์เท่าเดิมสองรอบ เพื่อให้แน่ใจว่าเขียนไฟล์จบแล้ว ทั้งไฟล์ถูกบันทึกในครั้งเดียว ถ้ามีจุดที่บันทึกไม่ได้ จะไม่เก็บครึ่งไฟล์

| ผล | สิ่งที่เกิดขึ้นกับไฟล์ |
|---|---|
| อ่านได้ | ย้ายไปโฟลเดอร์สำเร็จ |
| อ่านไม่ได้ | ย้ายไปโฟลเดอร์ผิดพลาด |

การย้ายไฟล์แปลว่าเก็บข้อมูลได้หรือไม่ได้ ยังไม่ใช่ผลตรวจสูตร

---

## เอาค่ามาเทียบกันอย่างไร

ระบบไล่ตามลำดับนี้ เรื่องไหนไม่ผ่านก็หยุด

**1. จับคู่**

```
MaterialCode ใน REPID          =  BulkCode ใน POST /api/prodprc
ProcessNo แถวแรกใน REPID      =  ProcessType ใน POST /api/prodprc
```

ถ้า `ProcessNo` ว่าง หรือยังไม่มีชุดที่ `BulkCode` กับ `ProcessType` ตรงกัน จะไม่มี `ValidationResult` ไม่ถือว่าไม่ผ่าน

ถ้ามีหลายชุดที่รหัสตรงกันหมด เลือกชุดที่วันที่มีผลใหม่สุด แล้วเลือก `Revision` มากสุด แล้วเลือกชุดที่เข้ามาหลังสุด

**2. โรงงาน**

```
Plant ใน POST /api/prodprc   ต้องอยู่ใน   BATCH_ALLOWED_PLANTS
```

**3. ขนาดล็อต**

```
BatchSizeMin  ≤  TargetQty  ≤  BatchSizeMax
```

ไม่ผ่านเรื่องนี้ `ValidationResult` เป็น `HEADER_MISMATCH`

**4. รหัสวัตถุดิบ**

ใช้เฉพาะขั้นที่ `param01` เป็น `RM Code` และ `param03` เป็น `%RM`

- รหัสที่มีปริมาณใน `REPID` ต้องอยู่ใน `param02` ของบางขั้น
- ทุกขั้นนั้นต้องมีอย่างน้อยหนึ่งรหัสใน `param02` ที่ไฟล์ `REPID` มีอยู่

ไม่ผ่านเรื่องนี้ `ValidationResult` เป็น `MATERIAL_CODE_MISMATCH` ข้อความขึ้นต้นว่า `Order code not in recipe:`

**5. ปริมาณ** ใช้เฉพาะขั้น `FIXED_PERCENT`

```
ปริมาณที่ต้องใช้ = TargetQty × param04 ÷ 100
ส่วนต่าง = RequirementQty − ปริมาณที่ต้องใช้
```

`RequirementQty` คือยอดรวมของรหัสนั้นในไฟล์ `REPID` ถ้าวางไฟล์เลขใบสั่งเดิมเข้ามากกว่าหนึ่งครั้ง ยอดจะถูกบวกทุกครั้ง

`TargetQty` มาจากไฟล์ `REPID` `param04` มาจาก `POST /api/prodprc`

| ส่วนต่าง | คำที่บันทึก | ความหมาย |
|---|---|---|
| ไม่เกิน 0.001 | `MATCH` | ตรง |
| น้อยกว่า −0.001 | `SHORTAGE` | ขาด |
| มากกว่า 0.001 | `EXCESS` | เกิน |

`G` หาร 1,000 เป็น `KG` ก่อนคิด แล้วคูณ 1,000 กลับตอนแสดงผล รหัสเดียวกันต้องใช้หน่วยเดียวกันทั้งไฟล์

ถ้ารหัสหนึ่งอยู่หลายขั้น ต้องบวกปริมาณที่ต้องใช้ของทุกขั้นนั้นก่อน แล้วค่อยลบกับ `RequirementQty` ครั้งเดียว

ขั้น `ADJUST` ไม่เข้าสูตรนี้ ตรวจแค่มีรหัสใน `param02`

มี `SHORTAGE` หรือ `EXCESS` แม้รหัสเดียว `ValidationResult` เป็น `QUANTITY_MISMATCH` ถ้าตรงทุกตัวเป็น `VALIDATED` และผลถูกล็อก

คู่ที่เคยมีผลแล้วจะไม่ถูกตรวจซ้ำ แม้จะเปลี่ยน `BATCH_ALLOWED_PLANTS` ภายหลัง ต้องลบผลเดิมก่อน

---

## ตัวอย่างที่ผ่าน `REPID_2026082811272843_2.csv`

ในไฟล์นี้ เลข `80000003` มีค่าดังนี้

| คำใน `REPID_2026082811272843_2.csv` | ค่า |
|---|---|
| `MaterialCode` | 206.0202 |
| `ProcessNo` | D-PC-2060202-Z-01 |
| `TargetQty` | 10000 |
| `TargetUnit` | KG |
| รหัส 120247 | `RequirementQty` 2200 หน่วย KG |
| รหัส 101003 | `RequirementQty` 209 หน่วย KG |
| รหัส 106020 | `RequirementQty` 20 หน่วย G |
| รหัส 108993 | `RequirementQty` 10 หน่วย KG |

ชุดที่ `BulkCode` และ `ProcessType` ตรงกัน ส่งเข้า `POST /api/prodprc` ไว้แล้ว ค่าหัวชุดคือ

| คำใน `POST /api/prodprc` | ค่า |
|---|---|
| `Plant` | HHN |
| `BulkCode` | 206.0202 |
| `BulkName` | FLC-PI(02) |
| `ProcessType` | D-PC-2060202-Z-01 |
| `Revision` | 1 |
| `BatchSizeMin` | 10000 |
| `BatchSizeMax` | 20000 |
| `Unit` | KG |

ตัวอย่างขั้นตอนที่ย่อจากของจริง 36 ขั้น เหลือเฉพาะขั้นที่ใช้ในตัวอย่าง

```json
{
  "Result": [
    {
      "Plant": "HHN",
      "BulkCode": "206.0202",
      "BulkName": "FLC-PI(02)",
      "ProcessType": "D-PC-2060202-Z-01",
      "Revision": "1",
      "BatchSizeMin": "10000",
      "BatchSizeMax": "20000",
      "Unit": "KG",
      "Process": [
        { "param01": "RM Code", "param02": "120226|120246|120247", "param03": "%RM", "param04": "22" },
        { "param01": "RM Code", "param02": "101003", "param03": "%RM", "param04": "2.09" },
        { "param01": "RM Code", "param02": "106020", "param03": "%RM", "param04": "0.0002" },
        { "param01": "RM Code", "param02": "108993|108717", "param03": "%RM", "param04": "0.03" },
        { "param01": "RM Code", "param02": "108993|108717", "param03": "%RM", "param04": "0.03" },
        { "param01": "RM Code", "param02": "108993|108717", "param03": "%RM", "param04": "0.04" }
      ]
    }
  ]
}
```

ลูกค้าได้ HTTP 200 ว่า `Production Process received successfully` และ `processStatus` เป็น `PENDING` ไฟล์ `REPID_2026082811272843_2.csv` ถูกย้ายไปโฟลเดอร์สำเร็จ

การเทียบของเลข `80000003`

| เรื่อง | `REPID_2026082811272843_2.csv` | `POST /api/prodprc` | สิ่งที่เกิดขึ้น |
|---|---|---|---|
| สินค้า | `MaterialCode` 206.0202 | `BulkCode` 206.0202 | ตรงกัน |
| กระบวนการ | `ProcessNo` D-PC-2060202-Z-01 | `ProcessType` D-PC-2060202-Z-01 | ตรงกัน |
| โรงงาน | ไม่ใช้เลข 30 ในไฟล์ | `Plant` HHN | อยู่ใน `BATCH_ALLOWED_PLANTS` |
| ขนาดล็อต | `TargetQty` 10000 | 10000 ถึง 20000 | อยู่ในช่วง |

ปริมาณ ตัวคูณ 10000 มาจาก `TargetQty` ในไฟล์ เปอร์เซ็นต์มาจาก `param04`

| รหัส | `param02` และ `param04` | คำนวณ | `RequirementQty` ในไฟล์ | ส่วนต่าง |
|---|---|---|---|---|
| 120247 | เลือกได้ 120226 หรือ 120246 หรือ 120247 ที่ 22 | 10000 × 22 ÷ 100 = 2200 KG | 2200 KG | 0 `MATCH` |
| 101003 | 2.09 | 10000 × 2.09 ÷ 100 = 209 KG | 209 KG | 0 `MATCH` |
| 106020 | 0.0002 | 10000 × 0.0002 ÷ 100 = 0.02 KG | 20 G = 0.02 KG | 0 `MATCH` |
| 108993 | สามขั้น 0.03, 0.03 และ 0.04 | 3 + 3 + 4 = 10 KG | 10 KG | 0 `MATCH` |

`param02` ของขั้น 22% มีสามรหัส ไฟล์นี้มีแค่ 120247 จึงใช้รหัสนั้น

ระบบบันทึก

- `ValidationResult` = `VALIDATED`
- `ValidationDetail` = All MaterialCode and Quantity matched

ข้อความนี้ไม่ได้ส่งกลับไปกับ HTTP 200 และไม่ได้ส่งกลับไปกับไฟล์

---

## ตัวอย่างที่ส่งไฟล์เดิมเข้าซ้ำ

วันที่ 12 ต.ค. 2569 ไฟล์ `REPID_2026082811272843_2.csv` ถูกวางเข้าอีกครั้ง

`MaterialCode`, `ProcessNo`, `TargetQty`, `param04` และ `Plant` ไม่เปลี่ยน สิ่งที่เปลี่ยนคือ `RequirementQty` เพราะยอดของเลขใบสั่งเดียวกันถูกบวกทุกครั้งที่ไฟล์เข้า

```
RequirementQty ของ 101003 = 209 + 209 = 418 KG
ปริมาณที่ต้องใช้ยัง = 10000 × 2.09 ÷ 100 = 209 KG
ส่วนต่าง = 418 − 209 = +209   →   EXCESS
```

อีกแปดรหัสเกินเป็นสองเท่าเหมือนกัน

- `ValidationResult` = `QUANTITY_MISMATCH`
- `ValidationDetail` = 9 material quantity mismatch(es)

ผล `VALIDATED` ของรอบแรกไม่ได้ถูกลบ แค่ไม่ใช่ผลปัจจุบัน

เลข `80000001` เกิดแบบเดียวกัน แต่คนละไฟล์ ไฟล์ `REPID_2026081311272843_2.csv` ให้ 209 กิโลกรัม และไฟล์ `REPID_2026082811272843_2.csv` ให้อีก 209 กิโลกรัม `ProcessNo` ของเลขนี้คือ `D-PC-2060202-B-01` ซึ่งตรงกับ `ProcessType` ของชุด `Plant` HHN อีกชุดหนึ่ง ผลก็เป็น `QUANTITY_MISMATCH` เก้าตัวเช่นกัน

---

## ตัวอย่างที่หยุดที่โรงงาน `REPID_TEST_PLANT_HH.csv`

ไฟล์นี้มีเลข `TEST-HH-01`

| คำใน `REPID_TEST_PLANT_HH.csv` | ค่า |
|---|---|
| `MaterialCode` | 206.0202 |
| `ProcessNo` | D-PC-2060202-A-05 |
| `TargetQty` | 100 |
| `TargetUnit` | KG |

ชุดที่รหัสตรงกันใน `POST /api/prodprc`

| คำ | ค่า |
|---|---|
| `BulkCode` | 206.0202 |
| `BulkName` | FLC-PI(02) |
| `ProcessType` | D-PC-2060202-A-05 |
| `Plant` | HH |
| `Revision` | 5 |
| `BatchSizeMin` | 3500 |
| `BatchSizeMax` | 10000 |

จับคู่ได้เพราะ `MaterialCode` ตรง `BulkCode` และ `ProcessNo` ตรง `ProcessType` จากนั้นเทียบ `Plant` HH กับ `BATCH_ALLOWED_PLANTS` ซึ่งมีแค่ `HHN` จึงหยุด `TargetQty` 100 ยังไม่ถูกเอาไปคูณ `param04`

- `ValidationResult` = `PLANT_NOT_ALLOWED`
- `ValidationDetail` = Plant HH is not allowed.

ถ้าภายหลังใส่ `HH` ใน `BATCH_ALLOWED_PLANTS` เลขนี้จะไปไม่ผ่านเรื่องถัดไป เพราะ `TargetQty` 100 น้อยกว่า `BatchSizeMin` 3500 และจะได้ `HEADER_MISMATCH`

ไฟล์เดียวกันมีเลข `TEST-HH-02` อีกใบ `MaterialCode` เป็น `20A.310` และ `ProcessNo` เป็น `D-PC-20A310-C-03` ชุดที่ตรงกันชื่อ `FLB-AF-BL` มี `Plant` เป็น `HH` เช่นกัน จึงได้ข้อความ `Plant HH is not allowed.` เหมือนกัน

---

## ตัวอย่างที่รหัสไม่อยู่ใน `param02`

เลข `81111113` อยู่ในไฟล์ `REPID_2026082811272843_2.csv`

| คำในไฟล์ | ค่า |
|---|---|
| `MaterialCode` | 205.M50 |
| `ProcessNo` | D-PC-205M50-B-02 |
| `TargetQty` | 10000 KG |
| รหัส 118003 | `RequirementQty` 18406 KG |

ชุดที่ตรงกันใน `POST /api/prodprc` ชื่อ `FLE-PLUS-GO` มี `Plant` เป็น `HHN` และรับล็อตได้ 3500 ถึง 10000 ดังนั้นเรื่องโรงงานกับขนาดล็อตผ่าน

`param02` ของชุดนี้มี 104067, 104090, 104064, 108975, 108972, 120217, 116057 ไม่มี 118003 และไม่มีขั้น `ADJUST` ของรหัสนี้

- `ValidationResult` = `MATERIAL_CODE_MISMATCH`
- `ValidationDetail` = Order code not in recipe: 118003

ยังไม่คิดส่วนต่างของรหัสอื่น รหัสอื่นในไฟล์นี้มีปริมาณมากกว่า `TargetQty × param04 ÷ 100` ประมาณเท่าตัว ถ้าไม่มี 118003 แล้วตรวจใหม่ น่าจะได้ `QUANTITY_MISMATCH`

---

## ตัวอย่างที่ไฟล์เข้าแล้วไม่มี `ValidationResult`

ไฟล์ชุดนี้ถูกย้ายไปโฟลเดอร์สำเร็จ แต่จับ `BulkCode` กับ `ProcessType` ไม่ได้ จึงไม่มีผลตรวจ

| ไฟล์ | เลขในไฟล์ | `MaterialCode` | `ProcessNo` | เหตุที่ไม่มีผล |
|---|---|---|---|---|
| `REPID_2026100909240012_1.csv` | 80280736 | 035.0A0 | ว่าง | ไม่มี `ProcessNo` |
| `REPID_2026100909240012_1.csv` | 80280737 | 031.HA01 | ว่าง | ไม่มี `ProcessNo` |
| `REPID_2026100909465064_1.csv` | 80278141 | 031.HA01 | ว่าง | ไม่มี `ProcessNo` |
| `REPID_2026100909514959_1.csv` | 80278152 | 031.HA01 | ว่าง | ไม่มี `ProcessNo` |
| `REPID_2026100909561731_1.csv` | 80271837 | 031.O20 | D-PC-031O20-4000-03 | ยังไม่มีชุดที่ `BulkCode` เป็น 031.O20 |
| `REPID_2026100910024657_1.csv` | 80269751 | 20A.190 | D-PC-20A190-C-03 | ยังไม่มีชุดที่ `BulkCode` เป็น 20A.190 |
| `REPID_2026100910164863_1.csv` | 80275886 | 20A.190 | D-PC-20A190-B-03 | ยังไม่มีชุดที่ `BulkCode` เป็น 20A.190 |

เมื่อมีชุดที่ `BulkCode` กับ `ProcessType` ตรงกันเข้ามาทาง `POST /api/prodprc` ระบบจะตรวจให้เองในรอบถัดไป ไม่ต้องวางไฟล์ `REPID` ใหม่

---

## อ่านผลที่เห็นในระบบ

| สิ่งที่เห็น | อ่านว่า |
|---|---|
| HTTP 200 และ `processStatus` เป็น `PENDING` | รับสูตรจาก `POST /api/prodprc` แล้ว ยังไม่ได้ตรวจใบสั่ง |
| ไฟล์ `REPID` ไปโฟลเดอร์สำเร็จ แต่ไม่มี `ValidationResult` | เก็บไฟล์แล้ว แต่ `ProcessNo` ว่าง หรือยังไม่มี `BulkCode` กับ `ProcessType` ที่ตรงกัน |
| `VALIDATED` | `MaterialCode` ตรง `BulkCode`, `ProcessNo` ตรง `ProcessType`, `Plant` ได้รับอนุญาต, `TargetQty` อยู่ในช่วง, รหัสใน `param02` ครบ และส่วนต่างไม่เกิน 0.001 |
| `PLANT_NOT_ALLOWED` | `Plant` ที่ส่งเข้า API ไม่อยู่ใน `BATCH_ALLOWED_PLANTS` |
| `HEADER_MISMATCH` | `TargetQty` อยู่นอก `BatchSizeMin` ถึง `BatchSizeMax` หรือหัวชุดไม่ครบ |
| `MATERIAL_CODE_MISMATCH` | มีรหัสใน `REPID` ที่ไม่มีใน `param02` |
| `QUANTITY_MISMATCH` | รหัสครบ แต่ `RequirementQty` ไม่เท่ากับ `TargetQty × param04 ÷ 100` มักเกิดเมื่อวางไฟล์เลขเดิมเข้ามากกว่าหนึ่งครั้ง |
| เปลี่ยน `BATCH_ALLOWED_PLANTS` แล้วผลเดิมไม่เปลี่ยน | คู่ที่ตรวจแล้วจะไม่ถูกตรวจซ้ำ |
