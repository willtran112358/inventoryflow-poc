# InventoryFlow Parts Extractor

> **Tiếng Việt (đọc nhanh):** bối cảnh dữ liệu + kiến trúc pipeline gộp một file — [`docs/DATA_AND_ARCHITECTURE_VI.md`](docs/DATA_AND_ARCHITECTURE_VI.md).

This project takes one big Excel file (`InventoryFlow_Data.xlsx`, ~230 MB) full
of ATV / pitbike / dirtbike parts catalog data and turns it into a clean set of
files that are easy to use in code or browse by hand:

- a SQLite database (`parts.db`) with one row per unique part number
- a folder of every embedded picture from the workbook (`images/`)
- two human-readable JSON files (`summary.json`, `sample_data.json`)

The goal is simple: take the Excel file, give us back tidy data we can build a
website, app, or dealer tool on top of, without anyone having to open the
spreadsheet again.

---

## What the script does, step by step

The script is a single Python file: `extract.py`. When you run it, it does
three things in order:

### Stage 1 — pull out the pictures and figure out where each one lives

An `.xlsx` file is really just a ZIP archive. Inside that archive every picture
that was pasted into the workbook is stored in `xl/media/` as a normal `.png`
or `.jpg`.

1. The script opens the workbook as a ZIP and copies every image file into a
   fresh `images/` folder.
2. Excel keeps a separate file (a "drawing") that says *which cell each picture
   is anchored to*. The script reads those drawings and builds a small lookup:
   "in sheet X, image `image1234.png` is anchored at row 47."

That lookup is what lets us link a picture back to a part later.

### Stage 2 — read every parts catalog sheet and pull out the part rows

The workbook has 110 sheets. Most of them are parts catalogs for a specific
ATV / bike model (for example `Bull125 AU125-2 (2021+)` or `KMB60 Engine`). A
few are reference tables (battery specs, spark plugs, owner's manuals, etc.)
that don't fit the part-number model — the script skips those.

For each parts sheet, the script:

1. Walks down the rows looking for a header row. Every catalog has a header
   like `No. | Part Number | EN name | CN name | … | Retail`, and many sheets
   repeat that header several times because each parts diagram (frame, brakes,
   engine top end, etc.) is its own little table within the sheet. The script
   recognizes a few different header styles used across the workbook
   (English, Chinese, "U8 Code", "NEW PART NUMBER", "Parts No.", etc.) and
   picks the best part-number column when a sheet has more than one.
2. Once it has a header, it reads each following row as a part: part number,
   English name, Chinese name, retail price.
3. The sheet name itself is the **fitment** — it tells us what model this part
   fits. If the sheet name contains a year range (`(2018-2020)`, `(2021+)`,
   `(2024)`, etc.) the script captures that too.
4. If a picture in this sheet was anchored at the same row as the part, the
   path to that picture gets attached to the part as `image_path`.
5. Parts that show up in more than one sheet (e.g. a bolt that fits ten
   different models) are stored once, with all the model fitments collected
   into a JSON list.

### Stage 3 — write the outputs

Once the run is finished:

- `parts.db` is created from scratch with the part rows.
- `summary.json` is written with the headline numbers and a per-sheet count.
- `sample_data.json` is written with the first 100 parts in pretty form so a
  non-technical reviewer can open it in Notepad.

---

## Database schema

The database is a single SQLite file (`parts.db`) with one table:

```sql
CREATE TABLE parts (
    part_number  TEXT PRIMARY KEY,   -- the manufacturer part number, unique
    english_name TEXT,               -- English description (e.g. "front brake assy")
    chinese_name TEXT,               -- Chinese description (e.g. "前碟刹总成")
    price        DECIMAL,            -- retail price in USD (from the "Retail" column)
    image_path   TEXT,               -- path to the picture, e.g. "images/image142.png"
    fitment      TEXT                -- JSON array — see below
);
```

| column         | what it means                                                                 |
| -------------- | ----------------------------------------------------------------------------- |
| `part_number`  | The unique part number. Used as the primary key.                              |
| `english_name` | The English part name as it appears in the workbook.                          |
| `chinese_name` | The Chinese name. Empty if the source sheet didn't have a Chinese column.     |
| `price`        | Retail price (the "Retail" column in the workbook). Empty if not listed.      |
| `image_path`   | Relative path to the part's picture. Empty if no picture was anchored to it.  |
| `fitment`      | JSON list of `{year, make, model, sheet}` records — every model this fits.    |

### Sample of the `fitment` JSON

The `fitment` column is a JSON list because the same part often fits many
models. Here is what it looks like for a common handlebar grip:

```json
[
  { "year": null,        "make": null, "model": "FOXStorm 70 AY70-2",      "sheet": "FOXStorm 70 AY70-2" },
  { "year": "2016-2020", "make": null, "model": "PREDATOR 125",            "sheet": "PREDATOR 125 (2016-2020)" },
  { "year": null,        "make": null, "model": "Storm150 A150",           "sheet": "Storm150 A150" },
  { "year": "2021+",     "make": null, "model": "Bull125 AU125-2",         "sheet": "Bull125 AU125-2 (2021+)" }
]
```

Each entry has four fields:

- **year** — the year range pulled from the sheet name when one is present
  (e.g. `"2016-2020"`, `"2021+"`, `"2024"`). `null` when the sheet name didn't
  include a year.
- **make** — always `null`. The workbook does not record a make column, and
  the sheet names use the model code, not a brand. If a brand needs to be
  attached later it would be a one-line lookup.
- **model** — the model name from the sheet, with the year part stripped out
  so it's clean (`"PREDATOR 125 (2016-2020)"` becomes `"PREDATOR 125"`).
- **sheet** — the original sheet name, kept verbatim so it's always traceable
  back to the source.

---

## Output files

After running the script you'll see this folder layout:

```
inventoryflow/
├── InventoryFlow_Data.xlsx     (the original spreadsheet, untouched)
├── extract.py                  (the script)
├── parts.db                    (SQLite database — 8,385 parts)
├── summary.json                (totals + sheet-by-sheet row counts)
├── sample_data.json            (first 100 full part records, pretty-printed)
├── ARCHITECTURE.md             (this document)
└── images/                     (1,586 picture files — image1.png, image2.jpg, …)
```

### `summary.json`

Top-level numbers and a list of every sheet with how many parts the script
pulled from it. Useful for sanity-checking that nothing was missed.

```json
{
  "total_parts": 8385,
  "total_sheets": 110,
  "images_extracted": 1586,
  "sheets": [
    { "name": "FOXStorm 70 AY70-2 ", "parts_extracted": 256 },
    ...
  ]
}
```

### `sample_data.json`

First 100 records in full, pretty-printed, so anyone can open it in Notepad
and see exactly what a part looks like.

---

## Python packages used (and why)

| package    | why we use it                                                              |
| ---------- | -------------------------------------------------------------------------- |
| `openpyxl` | Reads the Excel workbook cell-by-cell. Chosen over `pandas` because the   |
|            | file is 230 MB and the headers move around — we need row-level control.   |
| `pillow`   | Pre-installed alongside openpyxl; allows openpyxl to handle image cells   |
|            | without warnings. (We don't process images, just copy them.)              |
| stdlib     | Everything else (`zipfile`, `xml.etree`, `sqlite3`, `json`, `re`) is in   |
|            | the Python standard library — no extra installs needed.                   |

---

## How to run the script again in the future

1. Make sure `InventoryFlow_Data.xlsx` is in this folder (same name, exactly).
2. From a terminal in this folder, install the two packages once:

   ```
   py -m pip install openpyxl pillow
   ```

3. Run the script:

   ```
   py extract.py
   ```

   It takes about 1–2 minutes on a normal laptop. Running it again will
   overwrite `parts.db`, `images/`, `summary.json`, and `sample_data.json`
   — the original workbook is never modified.

If you replace `InventoryFlow_Data.xlsx` with a newer version that has the
same sheet style, just run the script again — it will rebuild everything
from the new file.

---

## Notes for future work

A few small things that would be obvious next steps if/when they're needed:

- **Make column** — the workbook has no make/brand column anywhere. If it's
  important downstream, the cleanest place to add it is a small lookup that
  maps each sheet name to a brand, applied while building the `fitment`
  entries.
- **Image-to-part linking** — about 655 of the 8,385 parts currently have a
  picture attached. The rest of the 1,586 pictures are saved in `images/` but
  not yet linked to a specific part. The matching happens by row anchor; some
  sheets place all their pictures above the parts table and the script can't
  tell which picture belongs to which part. Improving this would mean
  inspecting the visual layout of those specific sheets — worth doing only if
  pictures are needed for a specific use case.
- **Reference sheets** (battery specs, spark plug specs, owner's manuals,
  carb jets, etc.) are intentionally skipped because they don't fit the
  "one row = one part" shape. If we want them in the database later, they'd
  each go into their own table.
