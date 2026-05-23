# CLAUDE.md

Guidance for Claude Code working in this repo.

## What this is

A single-page web app showing registered massage/spa establishments in Chiang Mai. Two main views:

- **Public** (สำหรับประชาชน): 3 tabs
  - **ภาพรวม** — district + place dropdowns → jump to map
  - **แผนที่** — Leaflet map with markers for all currently-licensed places
  - **ค้นหา** — 3 text inputs (TH name, EN name, license) + อำเภอ/ตำบล filter dropdowns, results render as a scrollable **list** of cards (not a single result)
- **Staff** (สำหรับเจ้าหน้าที่): paginated CRUD table + JSON export. **Password-gated.**

Live at https://aunggrid.github.io/chiang-mai-spa-locator/ (GitHub Pages, main branch root). `git push` triggers a rebuild in ~1 minute.

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
- `processMapData()` clears existing markers before re-pinning so re-entries don't stack. Only `getStatus(spa).open === true` records get a marker; closed places are skipped on the public map.
- `populateOverviewDropdowns()` runs after data loads and feeds both the Overview tab and the Search tab's filter dropdowns from the same `uniqueAmphoes()` / `uniqueTambons(amphoe)` helpers.

## Column keys

The JS uses Thai column names verbatim — they match the JSON exactly. Constants are at the top of the script:

```
COL_NO, COL_NAME_TH, COL_NAME_EN, COL_LICENSE, COL_OWNER, COL_TYPE, COL_TEL,
COL_ADDRESS, COL_LIC_DATE, COL_EXPIRE, COL_NO_ADDR, COL_SOI, COL_ROAD,
COL_TAMBON ('เขต' = ตำบล), COL_AMPHOE ('เขวง' = อำเภอ — typo carried over from source Excel), COL_PROVINCE,
COL_LAT, COL_LNG
```

**Don't rename these to English** — the data file's keys are Thai and JS lookups depend on the exact strings. In particular, the typo `เขวง` (not `แขวง`) is load-bearing — if you "fix" it in either side without also fixing the other, the amphoe dropdown silently goes empty.

## Status logic

`getStatus(spa)` derives open/closed from `วันที่หมดอายุ` (ISO `YYYY-MM-DD` string):
- expiry ≥ today → "เปิดทำการ" (`--color-public`)
- expiry < today → "ปิดทำการ" (`--color-staff`)

## Staff password gate

- Entered via the existing landing-page "สำหรับเจ้าหน้าที่" button.
- `enterStaff()` checks `sessionStorage[STAFF_UNLOCK_KEY]` first; if absent, opens `#authModal` instead of `#staff`.
- `submitAuth()` hashes the input via Web Crypto (`crypto.subtle.digest('SHA-256')`) and string-compares against `STAFF_PASS_HASH` — a hex digest constant near the top of the staff section. The plaintext password is NOT in source.
- On success: sessionStorage flag set, modal closes, `doEnterStaff()` runs (which is the original entry logic — load data, render table).
- Unlock persists for the browser tab/session only.

To rotate the password: compute new SHA-256 hex digest (e.g. `python -c "import hashlib; print(hashlib.sha256(b'NEW').hexdigest())"`), replace `STAFF_PASS_HASH`, bump `STAFF_UNLOCK_KEY` to invalidate existing unlocks.

This is a casual gate, not real auth — it's bypassable from DevTools. If real access control is ever needed, the staff side has to move off a static site.

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
- The อำเภอ column header in the source is `เขวง` (typo). Do not "correct" it during export — the JS reads that exact key.

The full export script is in `README.md`.

## Color palette (from `spa-program.pdf`)

| Token | Hex | Used for |
|---|---|---|
| `--color-public` | `#51dda2` | safe / public CTA / status open / type badge / dropdown fill |
| `--color-staff`  | `#f08866` | staff CTA / status closed / delete button / password modal heading |
| `--color-warning`| `#f0aa93` | reset button |
| `--bg-color`     | `#efefef` | page background |
| `--text-dark`    | `#000000` | text / dark pill buttons |

## Things not to do

- Don't add a build system / framework. Single-file HTML is the explicit pattern.
- Don't reintroduce SheetJS — we migrated off it. The browser reads `data.json` only.
- Don't write to `data.json` from JS — browsers can't. CRUD lives in `localStorage`; staff uses **Export JSON** to download a fresh file then `git push` it.
- Don't translate Thai keys in `data.json`.
- Don't put the staff password plaintext anywhere in the repo (source, README, commit messages). Hash only.
- Don't revert the search panel back to a single `.find()` result — it intentionally lists all matches now.

## Common tasks

- **Add a form field**: edit `#formModal` markup, add a constant if the column is new, no other wiring needed (`saveForm` reads all named form elements).
- **Change page size**: `STAFF_PAGE_SIZE` near the top of the staff section.
- **Wipe local state during testing**: DevTools → Application → Local Storage → delete `spa_data_v1` (data) and `spa_staff_unlocked_v1` (staff session) — or click the staff page's Reset button (data only).
- **Rotate staff password**: see "Staff password gate" section.
- **Verify a change locally**: `python -m http.server 8000`, open `http://localhost:8000/`. For a quick UI screenshot, `chrome --headless=new --window-size=1440,900 --screenshot="$PWD/out.png" http://localhost:8000/`.
