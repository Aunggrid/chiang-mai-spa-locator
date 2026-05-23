# สถานประกอบการนวดสปา จังหวัดเชียงใหม่

เว็บแอปพลิเคชันแสดงตำแหน่งและจัดการข้อมูลสถานประกอบการสปา/เพื่อสุขภาพ ที่ขึ้นทะเบียนใน จ.เชียงใหม่ สนับสนุนโดย สาธารณสุขจังหวัดเชียงใหม่

**🌐 เว็บไซต์ (ใช้งานได้ทันที):** https://aunggrid.github.io/chiang-mai-spa-locator/

## คุณสมบัติ

### สำหรับประชาชน
3 แท็บหลัก:

1. **ภาพรวม** — เลือกอำเภอและสถานประกอบการจาก dropdown แล้วคลิกเพื่อกระโดดไปดูบนแผนที่ (เลือกได้ทั้งระดับร้านเดียวหรือทั้งอำเภอ)
2. **แผนที่** — แผนที่ interactive แสดงตำแหน่งสปาที่ขึ้นทะเบียนและ "เปิดทำการ" ทั้งหมด (ปักหมุดจาก Leaflet + OpenStreetMap) คลิกหมุดเพื่อดูชื่อ ที่อยู่ เบอร์โทร และลิงก์ไป Google Maps
3. **ค้นหา** — ค้นด้วย ชื่อ (ไทย / Eng) หรือ เลขใบอนุญาต กรองเพิ่มด้วยอำเภอ/ตำบล ผลลัพธ์แสดงเป็น **รายการ** ทุกร้านที่ตรงเงื่อนไข แต่ละการ์ดมีปุ่ม "ดูบนแผนที่" ของตัวเอง

แสดงสถานะ เปิด / ปิดทำการ จากวันหมดอายุใบอนุญาตอัตโนมัติ

### สำหรับเจ้าหน้าที่
- มี **password gate** ปกป้องหน้าจัดการ (รหัสผ่านสอบถามผู้ดูแลระบบ — ไม่ใส่ใน source สาธารณะ)
- เพิ่ม / แก้ไข / ลบ ข้อมูลสถานประกอบการ
- ค้นหาและแบ่งหน้า (25 รายการ/หน้า)
- Export ข้อมูลเป็น `data.json`
- รีเซ็ตกลับเป็นข้อมูลต้นฉบับ

> ⚠️ password เป็นเพียง casual gate ฝั่ง browser — ผู้ใช้ที่เปิด DevTools สามารถ bypass ได้ ถ้าต้องการความปลอดภัยจริงจังต้องมี backend แยก

## โครงสร้างโปรเจกต์

```
WebTungmap/
├── index.html          # แอปทั้งหมด (HTML + CSS + JS รวมในไฟล์เดียว)
├── data.json           # ข้อมูลสถานประกอบการ (1,699 รายการ)
├── สถานนวดสปา.xlsx     # ไฟล์ Excel ต้นฉบับ (2 sheet: data1 + data2)
├── spa-program.pdf     # เอกสาร design ต้นทาง
├── README.md
├── CLAUDE.md           # คู่มือสำหรับ AI assistant
└── .gitignore
```

## การพัฒนาแบบ Local

`fetch('data.json')` ถูกบล็อกเมื่อเปิดผ่าน `file://` ต้องเสิร์ฟผ่าน local web server:

```
python -m http.server 8000
```

เปิด http://localhost:8000/

หรือใช้ VS Code Live Server (คลิกขวาที่ `index.html` → Open with Live Server)

## การ Deploy

โปรเจกต์โฮสต์บน **GitHub Pages** (main branch root):

```
git add . && git commit -m "..." && git push
```

หลัง push ประมาณ 1 นาที GitHub จะ rebuild และเว็บที่ https://aunggrid.github.io/chiang-mai-spa-locator/ จะอัปเดต

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
3. กด **Export JSON** เพื่อดาวน์โหลด `data.json` ที่อัปเดต แล้ว commit + push ไฟล์ใหม่เข้า repo เพื่อให้ผู้ใช้คนอื่นเห็น
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

> 💡 ฟิลด์ของอำเภอใน Excel ต้นฉบับสะกดเป็น `เขวง` (ไม่ใช่ `แขวง`) — JS ฝั่งแอปอ้างคีย์ตามที่มีในไฟล์ ห้ามแก้ key ในระหว่าง export ไม่งั้น dropdown อำเภอจะกลายเป็นว่าง

## เทคโนโลยี

- HTML / CSS / Vanilla JavaScript (ไม่มี build step)
- [Leaflet 1.9.4](https://leafletjs.com/) สำหรับแผนที่
- [OpenStreetMap](https://www.openstreetmap.org/) สำหรับ map tiles
- [Google Fonts: Prompt](https://fonts.google.com/specimen/Prompt) ฟอนต์หลัก
- `localStorage` สำหรับ CRUD ฝั่ง browser
- `sessionStorage` สำหรับจำการปลดล็อก staff ใน tab เดียวกัน
- Web Crypto API (`crypto.subtle.digest`) สำหรับ hash รหัสผ่าน staff

## Color Palette (จาก `spa-program.pdf`)

| สี | Hex | ใช้สำหรับ |
|---|---|---|
| Safe (เขียว) | `#51dda2` | สำหรับประชาชน · สถานะเปิดทำการ · badge ประเภท · dropdown |
| Danger (ส้ม) | `#f08866` | สำหรับเจ้าหน้าที่ · สถานะปิดทำการ · ปุ่มลบ · password modal |
| Warning (ส้มอ่อน) | `#f0aa93` | ปุ่ม Reset |
| Background | `#efefef` | พื้นหลัง |
| Text | `#000000` | ตัวอักษร |

## License

ใช้ภายใต้การสนับสนุนของสาธารณสุขจังหวัดเชียงใหม่
