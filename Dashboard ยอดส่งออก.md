# Dashboard ยอดส่งออก

ไฟล์นี้เป็นคู่มือและพรอมป์มาตรฐานสำหรับสร้าง Dashboard ยอดส่งออกจากเว็บ Thailand Trade Report ของกระทรวงพาณิชย์ สำหรับสินค้าและ HS Code ใด ๆ ในอนาคต โดยผู้ใช้ควรแจ้งเพียง:

```text
ช่วยทำ Dashboard ยอดส่งออก [ชื่อสินค้า] HS Code [รหัส HS]
```

ตัวอย่าง:

```text
ช่วยทำ Dashboard ยอดส่งออก เครื่องดื่มอื่นๆ HS Code 220299
```

เมื่อได้รับคำสั่งลักษณะนี้ ให้สร้าง Dashboard รูปแบบเดียวกับงานต้นแบบ HS 220299: มีตัวเลข, กราฟ, filter, รายประเทศ, รายทวีป, รวมทุกประเทศ, growth, interaction เลือกจุดบนกราฟ, ตาราง sortable และเปิดใช้งานผ่าน localhost ก่อนส่งมอบ

## บทบาทการวิเคราะห์

ให้คิดแบบนักลงทุนและนักวางแผนกลยุทธ์:

- Dashboard ต้องช่วยดูทิศทางตลาดส่งออก, ประเทศ/ทวีปที่โตหรือหด, seasonality, concentration และ momentum
- ค่า default ต้องอ่านได้ทันทีโดยไม่ต้องกดอะไรเยอะ
- ตัวเลขต้อง reconcile กับแหล่งข้อมูลจริงก่อนส่งมอบ
- ถ้าเดือนล่าสุดของเว็บ MOC ไม่เท่ากับเดือนปัจจุบัน ให้ใช้เดือนล่าสุดที่เว็บ MOC ประกาศไว้ และระบุชัดเจนใน dashboard

## Input ขั้นต่ำ

ต้องการเพียง 2 อย่าง:

- `product_name`: ชื่อสินค้า เช่น `เครื่องดื่มอื่นๆ`
- `hs_code`: รหัส HS 6 หลัก เช่น `220299`

ค่า default:

- แหล่งข้อมูล: `https://tradereport.moc.go.th/th/stat/reporthscodeexport01`
- ช่วงเวลาเริ่มต้น: มกราคม 2564 (`2021-01`)
- ช่วงเวลาสิ้นสุด: เดือนล่าสุดตามหน้า MOC จากค่า `latestmonth` และ `latestyear`
- สกุลเงิน: บาท
- Grain หลัก: รายเดือน x ประเทศ
- Dashboard surface: static HTML + local Python HTTP server

ถาม follow-up เฉพาะเมื่อ:

- HS Code ไม่พบใน lookup ของ MOC
- Endpoint/API ของ MOC เปลี่ยนจนไม่สามารถดึงข้อมูลได้
- ผู้ใช้ต้องการช่วงเวลาหรือ destination ต่างจาก default อย่างชัดเจน

## Source และ API ที่ต้องใช้

เริ่มจากหน้า:

```text
https://tradereport.moc.go.th/th/stat/reporthscodeexport01
```

ขั้นตอน source discovery:

1. เปิดหรือ fetch หน้า report เพื่ออ่าน `latestmonth` และ `latestyear`
2. ตรวจ script/app bundle ถ้าจำเป็น เพื่อยืนยัน endpoint ปัจจุบัน
3. ใช้ endpoint หลัก:

```text
POST https://tradereport.moc.go.th/stat/reporthscodeexport01/result
```

4. ใช้ lookup ที่เกี่ยวข้อง:

```text
GET  https://tradereport.moc.go.th/lookup/years?lang=th
GET  https://tradereport.moc.go.th/lookup/country
GET  https://tradereport.moc.go.th/lookup/countrygroup?lang=th
GET  https://tradereport.moc.go.th/lookup/hscodeversions
POST https://tradereport.moc.go.th/lookup/hscodelist{HS_CODE}/{HS_VERSION}
```

ให้เริ่มจาก HS version ล่าสุดที่ใช้ได้กับเว็บปัจจุบัน เช่น `2022` ถ้า HS Code ไม่พบ ให้ลอง version ที่ endpoint `hscodeversions` ส่งกลับมา โดยต้องบันทึก version ที่ใช้ไว้ใน metadata

Payload สำหรับดึงข้อมูลรายเดือนควรมีโครงสร้างประมาณนี้:

```json
{
  "year": { "id": "2026", "text": "2569" },
  "month": { "id": "4", "text": "เม.ย." },
  "currency": { "id": "baht", "text": "บาท" },
  "hscodedigits": 6,
  "hscode": null,
  "sort": { "id": "value_desc", "text": "มูลค่า (จากมากไปน้อย)" },
  "hscodes": [{ "id": "220299", "name": "ชื่อสินค้าจาก lookup" }],
  "Previousyear": { "id": "0", "text": "0 ปี" },
  "hscodeversion": { "id": "2022", "text": "2022" },
  "lang": "th"
}
```

ให้เปลี่ยน `HS_CODE`, `HS_VERSION`, `year`, `month`, และชื่อสินค้าให้ตรงกับงานนั้น ๆ

## Data Pipeline

ให้สร้าง script ดึงข้อมูลแบบ reusable โดย parameterize `HS_CODE`, `PRODUCT_NAME`, `START_YEAR`, `START_MONTH`, `HS_VERSION`

ขั้นตอน:

1. สร้าง session พร้อม headers:
   - `Accept: application/json, text/plain, */*`
   - `Content-Type: application/json`
   - `Origin: https://tradereport.moc.go.th`
   - `Referer: https://tradereport.moc.go.th/th/stat/reporthscodeexport01`
   - browser-like `User-Agent`
2. อ่าน `latestyear/latestmonth` จากหน้า MOC
3. ดึง `years`, `country`, `countrygroup`, และ HS name จาก hscode lookup
4. สร้าง mapping ประเทศ -> ทวีป จาก `countrygroup?lang=th`
5. Loop ทุกเดือนตั้งแต่ `2021-01` ถึงเดือนล่าสุดของ MOC
6. POST payload รายเดือน
7. Save raw JSON ทุกเดือนลง `work/moc_raw`
8. Parse records:
   - `RowType == "S"` คือยอดรวมโลก/รวมทุกประเทศ
   - records อื่นคือรายประเทศ
   - ใช้ `ValueMonth` เป็นมูลค่ารายเดือน
   - ใช้ `QuantityMonth` เป็นปริมาณรายเดือน
   - `Value` และ `Quantity` ใน row summary เป็น YTD จาก source เก็บไว้ได้ แต่ dashboard หลักควร aggregate จาก monthly rows เพื่อ consistency
9. สร้าง dataset:
   - `monthly`: รายเดือน x ประเทศ
   - `totals`: รายเดือนรวมทุกประเทศจาก `RowType == "S"`
   - `continents`: lookup ทวีป
   - `countries`: lookup ประเทศพร้อมทวีป
   - `validation`: summary reconciliation

## Output Data Files

ให้เขียนไฟล์ไว้ใต้ `outputs/dashboard`:

```text
outputs/dashboard/index.html
outputs/dashboard/styles.css
outputs/dashboard/app.js
outputs/dashboard/data.js
outputs/dashboard/data/dataset.json
outputs/dashboard/data/monthly_country_hs{HS_CODE}.csv
outputs/dashboard/data/monthly_continent_hs{HS_CODE}.csv
outputs/dashboard/data/monthly_total_hs{HS_CODE}.csv
outputs/dashboard/data/validation_reconciliation.csv
```

`data.js` ต้อง expose:

```javascript
window.MOC_EXPORT_DATA = { ...dataset };
```

เพื่อให้ dashboard เปิดได้แบบ static ผ่าน localhost โดยไม่ต้องเรียก API ซ้ำทุกครั้ง

## Validation Gate

ห้ามถือว่างานเสร็จจนกว่าจะ validate:

1. จำนวนเดือนที่ดึง = จำนวนเดือนตั้งแต่ `2021-01` ถึง `latestPeriod`
2. ทุกเดือนต้องมี world summary row (`RowType == "S"`)
3. Reconcile รายเดือน:
   - `worldValue` ต้องเท่ากับผลรวม `ValueMonth` ของทุกประเทศ
   - `worldQuantity` ต้องเท่ากับผลรวม `QuantityMonth` ของทุกประเทศ
   - บันทึก diff ลง `validation_reconciliation.csv`
4. `maxAbsValueDiff` และ `maxAbsQuantityDiff` ควรเป็น `0` หรือใกล้ `0` จาก rounding เท่านั้น
5. ถ้ามีประเทศที่ map ทวีปไม่ได้ ให้ใส่ `UNMAPPED/ไม่จัดกลุ่ม` และระบุใน validation
6. ถ้า endpoint คืนข้อมูลว่าง ให้หยุดและรายงานว่า source blocked/invalid ไม่สร้าง mock data

## Metric Definitions

ให้คำนวณจาก monthly rows:

### มูลค่า

`value = sum(ValueMonth)`

หน่วย: บาท

### ปริมาณ

`quantity = sum(QuantityMonth)`

หน่วย: หน่วยตามกรมศุลกากร/MOC source

### รายเดือน

ใช้ค่าจากเดือนนั้นตรง ๆ

### รายไตรมาส

รวมเดือนในไตรมาสเดียวกัน:

```text
Q1 = Jan-Mar
Q2 = Apr-Jun
Q3 = Jul-Sep
Q4 = Oct-Dec
```

ถ้าไตรมาสล่าสุดยังไม่ครบ 3 เดือน ให้ tag เป็น `partial` และไม่คำนวณ QoQ/YoY ของไตรมาสนั้น เว้นแต่ผู้ใช้ระบุว่าต้องการเทียบแบบ YTD quarter-to-date

### รายปี

รวมทุกเดือนในปีนั้น

ถ้าปีล่าสุดยังไม่ครบ 12 เดือน ให้ tag เป็น `partial` และไม่คำนวณ YoY แบบเต็มปีในตาราง/กราฟรายปี แต่ KPI สามารถแสดง YTD YoY ได้โดยเทียบเดือนสะสมเท่ากันกับปีก่อน

### Share

```text
share_value_pct = entity_value / total_world_value_same_period * 100
share_quantity_pct = entity_quantity / total_world_quantity_same_period * 100
```

### Growth

```text
MoM = (current_month - previous_month) / previous_month * 100
YoY รายเดือน = (current_month - same_month_previous_year) / same_month_previous_year * 100
QoQ รายไตรมาส = (current_quarter - previous_quarter) / previous_quarter * 100
YoY รายไตรมาส = (current_quarter - same_quarter_previous_year) / same_quarter_previous_year * 100
YoY รายปี = (current_year - previous_year) / previous_year * 100
```

ถ้าค่าฐานเป็น `0`, missing, หรือ period ยังไม่ครบ ให้แสดง `-`

## Dashboard Layout

ให้ layout เป็น dashboard ใช้งานจริง ไม่ใช่ landing page

### Header

- Eyebrow: `Thailand Export Monitor`
- H1: `[PRODUCT_NAME] HS [HS_CODE]`
- Coverage text: `[HS_CODE] : [HS_NAME] | 2021-01 ถึง [latestPeriod]`
- Source freshness: `latest source month: [เดือนล่าสุด MOC] [ปีไทย]`
- ปุ่ม:
  - `MOC Source`
  - `CSV`

### KPI Cards

แสดง 6 card:

1. มูลค่าเดือนล่าสุด
2. ปริมาณเดือนล่าสุด
3. MoM มูลค่าเดือนล่าสุด
4. YoY มูลค่าเดือนล่าสุด
5. YTD มูลค่า พร้อม YTD YoY
6. YTD ปริมาณ พร้อม YTD YoY

### Global Filters

ต้องมี:

- ช่วงเวลา:
  - รายเดือน
  - รายไตรมาส
  - รายปี
- มุมมอง:
  - รวมทุกประเทศ
  - รายประเทศ
  - รายทวีป
- Metric:
  - มูลค่า
  - ปริมาณ
- Growth:
  - รายเดือน: MoM, YoY
  - รายไตรมาส: QoQ, YoY
  - รายปี: YoY

### Entity Filter

สำหรับรายประเทศ/รายทวีป:

- Search
- Top 10
- All
- Clear
- Checkbox list

Default:

- รายประเทศ: Top 10 ตามมูลค่าเดือนล่าสุดหรือมูลค่ารวม
- รายทวีป: เลือกทุกทวีป
- รวมทุกประเทศ: ซ่อน entity filter

## Charts

ต้องมี 2 chart หลัก:

### 1. กราฟยอดส่งออก

Line chart แสดง level ของมูลค่าหรือปริมาณ ตาม filter ปัจจุบัน

ต้องรองรับ:

- รายเดือน/รายไตรมาส/รายปี
- รวมทุกประเทศ/รายประเทศ/รายทวีป
- มูลค่า/ปริมาณ
- หลาย series พร้อม legend
- จุดบนเส้นกราฟต้อง clickable และ keyboard accessible

เมื่อกดจุด ให้แสดง detail panel ใต้กราฟ:

- งวด
- ประเทศ/ทวีป/รวมทุกประเทศ
- มูลค่า
- ปริมาณ
- Share มูลค่า
- Share ปริมาณ
- MoM มูลค่า
- MoM ปริมาณ
- YoY มูลค่า
- YoY ปริมาณ
- QoQ มูลค่า
- QoQ ปริมาณ

### 2. กราฟการเติบโต

Line chart แสดง growth ที่เลือก เช่น MoM, YoY, QoQ ของมูลค่าหรือปริมาณ

ต้องรองรับ:

- จุดบนเส้นกราฟ clickable และ keyboard accessible
- Highlight จุดที่เลือก
- Detail panel ใต้กราฟเหมือนกราฟยอดส่งออก
- แกน Y เป็นเปอร์เซ็นต์
- มี zero line เมื่อช่วงข้อมูลมีทั้งบวกและลบ

## ตารางตัวเลขรายงวด

ตารางต้องแสดง:

- งวด
- รายการ
- มูลค่า
- ปริมาณ
- Share
- Growth มูลค่า ตาม growth filter ปัจจุบัน
- Growth ปริมาณ ตาม growth filter ปัจจุบัน

ต้องมี controls:

- Rows: 50, 100, 250, All
- เรียงตาม:
  - งวด
  - มูลค่า
  - ปริมาณ
  - MoM มูลค่า
  - MoM ปริมาณ
  - YoY มูลค่า
  - YoY ปริมาณ
  - QoQ มูลค่า
  - QoQ ปริมาณ
- ลำดับ:
  - มากไปน้อย
  - น้อยไปมาก

Rules:

- รายเดือน: เปิดให้เรียง MoM และ YoY, ปิด QoQ
- รายไตรมาส: เปิดให้เรียง QoQ และ YoY, ปิด MoM
- รายปี: เปิดให้เรียง YoY, ปิด MoM/QoQ
- ค่า null หรือ `-` ต้องถูกดันไปท้ายตาราง
- CSV export ต้อง export ตารางตาม filter/sort ปัจจุบัน

## Source & Validation Section

ต้องแสดง:

- Source
- Endpoint
- Fetched UTC
- Coverage
- HS Code + HS Version
- จำนวน rows
- Reconciliation max diff ของ value และ quantity
- หมายเหตุ partial ของไตรมาส/ปีล่าสุด

## Frontend Rules

ใช้ static HTML/CSS/JS ได้ โดยไม่ต้องใช้ framework ถ้าไม่จำเป็น

Design:

- dashboard ต้องเป็น first screen ไม่ใช่ landing page
- ใช้ layout แบบ utilitarian สำหรับนักลงทุน
- ไม่มี hero marketing
- มี KPI, filter, charts, table, source section
- รองรับ desktop และ mobile
- ไม่ให้ text ล้น container
- chart ต้องไม่ blank
- กราฟต้องมี stable dimensions
- ใช้สีไม่ one-note เกินไป แต่ให้คงความสุขุม
- cards ใช้ border radius ไม่เกิน 8px

Interaction:

- จุดกราฟต้องมี `role="button"` หรือ semantic ที่กดได้
- รองรับ click และ keyboard Enter/Space
- มี `<title>` หรือ tooltip/accessible label สำหรับจุดกราฟ
- เมื่อเปลี่ยน filter ต้อง refresh chart, detail panel, และ table ให้สอดคล้องกัน

## Localhost Run

หลัง build แล้วให้เปิด localhost:

```powershell
python -m http.server 8776 --bind 127.0.0.1
```

ถ้า port ถูกใช้แล้ว ให้เลือก port ถัดไปที่ว่าง เช่น `8777`, `8780`

ส่งมอบ URL:

```text
http://127.0.0.1:{PORT}/
```

ถ้าใช้ Windows แล้วเจอ `CreateProcessAsUserW failed: 5` ให้ลองใช้ PowerShell path ตรง:

```powershell
C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NoProfile -Command "..."
```

สำหรับ output ภาษาไทยจาก Python ให้ใช้:

```powershell
python -X utf8 script.py
```

## QA Checklist

ก่อน final ต้องตรวจ:

1. `node --check outputs/dashboard/app.js`
2. `curl http://127.0.0.1:{PORT}/` ต้องได้หน้า dashboard
3. เปิดจริงด้วย browser/headless browser และจับ screenshot:
   - desktop ประมาณ `1440x1200`
   - mobile ประมาณ `390x1600` หรือสูงกว่า
4. ตรวจว่ากราฟ render ไม่ blank
5. ตรวจว่า text ไม่ล้นบน mobile
6. ตรวจว่า point detail panel แสดงหลังเลือกจุด
7. ตรวจว่า table sort controls แสดงและไม่ล้น
8. ตรวจ DOM ว่ามีจุด clickable เช่น `data-chart-point`, `role="button"`, `tabindex="0"`
9. ตรวจ `validation_reconciliation.csv`
10. สรุป coverage/latest source month ให้ชัดเจนใน final

## Final Response Template

ตอบผู้ใช้เป็นภาษาไทยแบบสั้น กระชับ:

```text
เสร็จแล้วครับ เปิด dashboard ได้ที่:

http://127.0.0.1:{PORT}/

ข้อมูลดึงจาก MOC สำหรับ HS {HS_CODE} ครอบคลุม {START_PERIOD} ถึง {LATEST_PERIOD}
เดือนล่าสุดตาม source คือ {LATEST_MONTH_TH} {LATEST_YEAR_TH}

ตรวจแล้ว:
- ดึงข้อมูลครบ {N_MONTHS} เดือน
- รายประเทศ {N_ROWS} rows
- reconciliation max value diff = {VALUE_DIFF}, max quantity diff = {QTY_DIFF}
- ทดสอบหน้า desktop/mobile แล้ว

ไฟล์อยู่ที่ outputs/dashboard/index.html
```

ถ้า source blocked หรือ validation ไม่ผ่าน ต้องบอกตรง ๆ ว่ายังไม่ complete และระบุ blocker

## Reference Implementation จากงานต้นแบบ

งานต้นแบบ:

```text
สินค้า: เครื่องดื่มอื่นๆ
HS Code: 220299
Coverage: 2021-01 ถึง 2026-04
Dashboard folder: C:\Users\Test\Documents\Codex\2026-06-10\dashboard-1-https-tradereport-moc-go\outputs\dashboard
Fetcher: C:\Users\Test\Documents\Codex\2026-06-10\dashboard-1-https-tradereport-moc-go\work\fetch_moc_hs220299.py
```

ให้นำ pattern นี้ไปทำซ้ำ โดยเปลี่ยน parameter ของสินค้าและ HS Code แทนการเขียนใหม่ทั้งหมด

## Minimal Future Prompt สำหรับผู้ใช้

ผู้ใช้สามารถสั่งแบบนี้:

```text
ทำ Dashboard ยอดส่งออก [ชื่อสินค้า] HS Code [HS_CODE]
```

Agent ต้องเข้าใจว่าให้ทำครบทุกขั้นตอนในไฟล์นี้โดยอัตโนมัติ:

- ดึงข้อมูล MOC ตั้งแต่ ม.ค. 2564 ถึงเดือนล่าสุด
- แยกประเทศ/ทวีป/รวมทุกประเทศ
- คำนวณรายเดือน/ไตรมาส/ปี
- คำนวณมูลค่าและปริมาณ
- คำนวณ MoM/YoY/QoQ
- ทำกราฟ interactive และตาราง sortable
- validate reconciliation
- เปิด localhost และส่งมอบ URL
