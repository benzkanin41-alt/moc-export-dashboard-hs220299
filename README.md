# HS 220299 Export Dashboard

Static GitHub Pages dashboard for Thailand export statistics from the Ministry of Commerce Trade Report.

## Dashboard

- Product: เครื่องดื่มอื่นๆ
- HS Code: 220299
- Source: https://tradereport.moc.go.th/th/stat/reporthscodeexport01
- Coverage: 2021-01 to 2026-05
- Latest source month: พ.ค. 2569
- Metrics: export value and quantity
- Breakdowns: total, country, continent
- Periods: monthly, quarterly, yearly
- Growth: MoM, YoY, QoQ where applicable

## Files

- `index.html` - dashboard page
- `styles.css` - dashboard styling
- `app.js` - dashboard interaction and calculations
- `data.js` - embedded dashboard dataset
- `data/` - JSON and CSV source extracts
- `Dashboard ยอดส่งออก.md` - reusable build prompt and implementation manual for future HS code dashboards

## Validation

The build reconciles country-level rows to the MOC world summary row by month.

Current validation from the generated dataset:

- Months fetched: 65
- Country-month rows: 7,493
- Max value diff: 0
- Max quantity diff: 0

## Local Run

```powershell
python -m http.server 8776 --bind 127.0.0.1
```

Then open:

```text
http://127.0.0.1:8776/
```
