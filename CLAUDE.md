# CLAUDE.md

Guidance for Claude Code working in this repo.

## What this is

A single-page web app showing registered massage/spa establishments in Chiang Mai. Two views:
- **Public** (สำหรับประชาชน): Leaflet map + 3-field search.
- **Staff** (สำหรับเจ้าหน้าที่): paginated table with CRUD + JSON export.

## Architecture

**Everything lives in `index.html`** — HTML + inline CSS + inline `<script>`, no build step, no framework. Don't introduce one without being asked.

```
data.json  →  Store (localStorage 'spa_data_v1')  →  spaData (in-memory)  →  UI
                          ↑                                  ↓
                          └─ Store.save() after CRUD ────────┘
```

- `data.json` is the seed file (UTF-8, 1699 records, indented JSON).
- `Store.load()` returns localStorage if present, else fetches `data.json` and seeds it.
- All mutations go through `Store.save(spaData)`.
- `processMapData()` clears existing markers before re-pinning so re-entries don't stack.

## Column keys

The JS uses Thai column names verbatim — they match the JSON exactly. Constants are at the top of the script:

```
COL_NO, COL_NAME_TH, COL_NAME_EN, COL_LICENSE, COL_OWNER, COL_TYPE, COL_TEL,
COL_ADDRESS, COL_LIC_DATE, COL_EXPIRE, COL_NO_ADDR, COL_SOI, COL_ROAD,
COL_TAMBON ('เขต' = ตำบล), COL_AMPHOE ('เขวง' = อำเภอ — typo carried over from source Excel), COL_PROVINCE,
COL_LAT, COL_LNG
```

**Don't rename these to English** — the data file's keys are Thai and JS lookups depend on the exact strings.

## Status logic

`getStatus(spa)` derives open/closed from `วันที่หมดอายุ` (ISO `YYYY-MM-DD` string):
- expiry ≥ today → "เปิดทำการ" (`--color-public`)
- expiry < today → "ปิดทำการ" (`--color-staff`)

## Running locally

`fetch('data.json')` is blocked under `file://`. Always serve via HTTP:

```
python -m http.server 8000
```

Then open http://localhost:8000/.

## Excel re-export gotcha

`สถานนวดสปา.xlsx` has two sheets (`data1`, `data2`). When regenerating `data.json`:
- Iterate `wb.sheetnames` and merge.
- Convert `datetime` cells to ISO strings (`YYYY-MM-DD`) — the browser status logic expects them parseable by `new Date()`.
- Filter rows that are entirely empty.
- Thai column names come back as proper Unicode (codepoints `U+0E00`–`U+0E7F`). If they look like mojibake in Windows `cmd.exe`, that's the CP874 codepage display, **not** corrupt bytes — read the file in Python with `encoding='utf-8'` to verify.

The full export script is in `README.md`.

## Color palette (from `theme.pdf`)

| Token | Hex | Used for |
|---|---|---|
| `--color-public` | `#51dda2` | safe / public CTA / status open / type badge |
| `--color-staff`  | `#f08866` | staff CTA / status closed / delete button |
| `--color-warning`| `#f0aa93` | reset button |
| `--bg-color`     | `#efefef` | page background |
| `--text-dark`    | `#000000` | text |

## Things not to do

- Don't add a build system / framework. Single-file HTML is the explicit pattern.
- Don't reintroduce SheetJS — we migrated off it. The browser reads `data.json` only.
- Don't write to `data.json` from JS — browsers can't. CRUD lives in `localStorage`; staff uses **Export JSON** to download a fresh file they replace manually.
- Don't translate Thai keys in `data.json`.

## Common tasks

- **Add a form field**: edit `#formModal` markup, add a constant if the column is new, no other wiring needed (`saveForm` reads all named form elements).
- **Change page size**: `STAFF_PAGE_SIZE` near the top of the staff section.
- **Wipe local state during testing**: DevTools → Application → Local Storage → delete `spa_data_v1`, or click the staff page's Reset button.
