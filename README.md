# สถานประกอบการนวดสปา จังหวัดเชียงใหม่

เว็บแอปพลิเคชันแสดงตำแหน่งและจัดการข้อมูลสถานประกอบการสปาที่ขึ้นทะเบียนใน จ.เชียงใหม่ สนับสนุนโดย สาธารณสุขจังหวัดเชียงใหม่

## คุณสมบัติ

### สำหรับประชาชน
- แผนที่ interactive แสดงตำแหน่งสปาที่ขึ้นทะเบียน
- ค้นหาด้วยชื่อ (ไทย / Eng) หรือเลขใบอนุญาต
- ดูรายละเอียดและเปิดทางใน Google Maps
- แสดงสถานะ เปิด / ปิดทำการ จากวันหมดอายุใบอนุญาต

### สำหรับเจ้าหน้าที่
- เพิ่ม / แก้ไข / ลบ ข้อมูลสถานประกอบการ
- ค้นหาและแบ่งหน้า (25 รายการ/หน้า)
- Export ข้อมูลเป็น `data.json`
- รีเซ็ตกลับเป็นข้อมูลต้นฉบับ

## โครงสร้างโปรเจกต์

```
WebTungmap/
├── index.html          # แอปทั้งหมด (HTML + CSS + JS รวมในไฟล์เดียว)
├── data.json           # ข้อมูลสถานประกอบการ (1,699 รายการ)
├── สถานนวดสปา.xlsx     # ไฟล์ Excel ต้นฉบับ (2 sheet)
├── theme.pdf           # เอกสาร design
├── README.md
└── CLAUDE.md           # คู่มือสำหรับ AI assistant
```

## การใช้งาน

ต้องเปิดผ่าน local web server (เปิดด้วย `file://` โดยตรง fetch ไม่ทำงาน)

### Python (มาพร้อม Windows ส่วนใหญ่)
```
python -m http.server 8000
```
เปิด http://localhost:8000/

### VS Code Live Server
ติดตั้งส่วนขยาย Live Server แล้วคลิกขวาที่ `index.html` → Open with Live Server

## การจัดการข้อมูล

ข้อมูลไหลผ่าน 3 ชั้น:

```
data.json  →  localStorage (spa_data_v1)  →  หน้าจอ
                    ↑                              ↓
                    └────── CRUD โดยเจ้าหน้าที่ ──────┘
                                  ↓
                          Export JSON (ดาวน์โหลดไฟล์ใหม่)
```

1. ครั้งแรกที่เปิด แอปจะ fetch `data.json` มา seed ลง `localStorage`
2. การ เพิ่ม / แก้ / ลบ จะมีผลใน `localStorage` ทันที (อยู่เฉพาะใน browser ของผู้ใช้คนนั้น)
3. กด **Export JSON** เพื่อดาวน์โหลด `data.json` ที่อัปเดต แล้วนำไปแทนที่ไฟล์เดิมในโฟลเดอร์เพื่อบันทึกถาวร
4. กด **Reset** เพื่อล้าง `localStorage` และโหลด `data.json` ต้นฉบับใหม่

## การ Export ข้อมูลจาก Excel ต้นฉบับ

ถ้ามีการอัปเดต `สถานนวดสปา.xlsx` ใหม่ ให้รัน:

```python
import openpyxl, json, datetime

wb = openpyxl.load_workbook('สถานนวดสปา.xlsx', data_only=True)
records = []
for sheet_name in wb.sheetnames:
    rows = list(wb[sheet_name].iter_rows(values_only=True))
    headers = rows[0]
    for row in rows[1:]:
        rec = {}
        for h, v in zip(headers, row):
            if not h: continue
            if isinstance(v, datetime.datetime):
                rec[h] = v.strftime('%Y-%m-%d')
            else:
                rec[h] = "" if v is None else v
        if any(rec.get(k) for k in rec):
            records.append(rec)

with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(records, f, ensure_ascii=False, indent=2)
```

## เทคโนโลยี

- HTML / CSS / Vanilla JavaScript (ไม่มี build step)
- [Leaflet 1.9.4](https://leafletjs.com/) สำหรับแผนที่
- [OpenStreetMap](https://www.openstreetmap.org/) สำหรับ map tiles
- [Google Fonts: Prompt](https://fonts.google.com/specimen/Prompt) ฟอนต์หลัก
- `localStorage` สำหรับ CRUD ฝั่ง browser

## Color Palette (จาก `theme.pdf`)

| สี | Hex | ใช้สำหรับ |
|---|---|---|
| Safe (เขียว) | `#51dda2` | สำหรับประชาชน · สถานะเปิดทำการ · badge ประเภท |
| Danger (ส้ม) | `#f08866` | สำหรับเจ้าหน้าที่ · สถานะปิดทำการ · ปุ่มลบ |
| Warning (ส้มอ่อน) | `#f0aa93` | ปุ่ม Reset |
| Background | `#efefef` | พื้นหลัง |
| Text | `#000000` | ตัวอักษร |

## License

ใช้ภายใต้การสนับสนุนของสาธารณสุขจังหวัดเชียงใหม่
