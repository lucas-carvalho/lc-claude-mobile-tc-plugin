# Spreadsheet Formatting Reference — xlsxwriter

## General File Structure

- **Main sheet**: `"Test Cases"` — contains all TCs
- **Secondary sheet**: `"Block Summary"` — TC count per block with color legend
- Freeze panes at row 2 (rows 0–1 frozen: title + headers)

## Columns (A–H)

| Col | Name | Width | Alignment | Notes |
|-----|------|-------|-----------|-------|
| A | Passed ✓ | 12 | Center | Boolean `False` — user converts to checkbox via Insert > Checkbox in GSheets |
| B | ID | 9 | Center | |
| C | Block | 36 | Left | |
| D | Scenario Title | 22 | Left | |
| E | Description (Given / When / Then) | 58 | Left | Gherkin format, wrap enabled |
| F | Notes / Mock | 32 | Left | Mock data, AC ref, gaps |
| G | Status | 16 | Center | Dropdown: Not tested / Passed / Failed / Blocked / N/A |
| H | Automatable? | 18 | Center | Dropdown: Yes / No / - (wider to fit label on one line) |

## Special Rows

- **Row 0** (xlsx row 1): Sheet title — merged A:H, blue `1A56A0`, white bold 14pt, height 38
- **Row 1** (xlsx row 2): Column headers — dark `1F2937`, white bold 11pt, height 28
- **TC rows**: NO `set_row()` call — Google Sheets auto-expands wrapped rows
- **Block separator rows**: merged A:H, block color, white bold 11pt, height 22
- **Empty separator row** (between last TC and Mock block): no writes, no `set_row()` — completely unstyled
- **Mock rows**: NO `set_row()` call — auto-expand (Key Variables may wrap)
- **AC Coverage rows**: NO `set_row()` call — auto-expand

## AC Coverage Layout

Sub-header and data rows use this column structure:

| A | B:C (merged) | D | E:H (merged) |
|---|---|---|---|
| AC | Description | Covered by | Status |

- G and H are absorbed into the Status merge (E:H) — no empty styled cells in G/H
- Status value: `"Covered"` (green `166534` on `D1FAE5`) or `"Not covered"` (red `7C2D12` on `FFEDD5`)

## Mock Key Variables Format

Use `\n` to separate the "Used by" list from the actual key variable descriptions:

```python
MOCKS = [
    ("Non-SCORM lesson in active enrollment",
     "Used by: TC-04, TC-05, TC-06\nlessonOrExamObj.isScorm: false, enrollmentId: [valid int], courseId: [valid int]"),
]
```

The Key Variables cell has `text_wrap=True` (default) and no fixed height → auto-expands.

## Google Sheets Compatibility Rules

1. **Column A checkbox**: Write Python `False` as a boolean — no DataValidation. In Google Sheets the user selects column A → Insert > Checkbox.

2. **Row heights**: Do NOT call `ws.set_row()` on TC rows, Mock rows, or AC Coverage rows. Without a fixed height, Google Sheets auto-expands wrapped rows.

3. **Data validation scope**: Apply Status (G) and Automatable (H) dropdowns **only to TC rows**. Compute the 0-indexed last TC row as `tc_last_0i = 2 + len(BLOCKS) + len(TEST_CASES) - 1` and use that as the upper bound. The empty separator row and Mock/AC rows are entirely outside the validation range.

4. **xlsxwriter format limit**: Create all formats via `wb.add_format()` before calling `wb.close()`. Pre-create per-block formats outside loops (see template).

## Data Validations

```python
# tc_last_0i: 0-indexed last TC row (inclusive)
tc_last_0i = 2 + len(BLOCKS) + len(TEST_CASES) - 1

ws.data_validation(2, 6, tc_last_0i, 6, {   # column G (0-indexed col 6)
    "validate": "list",
    "source": ["Not tested", "Passed", "Failed", "Blocked", "N/A"],
    "show_error": False,
})
ws.data_validation(2, 7, tc_last_0i, 7, {   # column H (0-indexed col 7)
    "validate": "list",
    "source": ["Yes", "No", "-"],
    "show_error": False,
})
```

## Block Color Palette

```python
# "Block Name": ("header_hex", "row_tint_hex")
COLOR_PALETTE = {
    "access":        ("1A56A0", "DDEEFF"),
    "structure":     ("92400E", "FEF3C7"),
    "ni_v1":         ("7B2D8B", "F0E0FF"),
    "ni_v2":         ("166534", "D1FAE5"),
    "ni_v3":         ("065666", "CCFBF1"),
    "ip_v1":         ("3D1A78", "EDE9FE"),
    "ip_v2":         ("1E3A5F", "DBEAFE"),
    "ip_v3":         ("14532D", "DCFCE7"),
    "completed":     ("7C2D12", "FFEDD5"),
    "types":         ("374151", "F3F4F6"),
    "edge":          ("1E3A8A", "EFF6FF"),
    "segment":       ("065666", "CCFBF1"),
    "firebase":      ("92400E", "FEF3C7"),
    "revenuecat":    ("4C1D95", "EDE9FE"),
    "nonfunc":       ("1E4D2B", "D1FAE5"),
    "mock":          ("374151", "F9FAFB"),
    "coverage":      ("1F2937", "F3F4F6"),
}
```

## Complete Python Script Template

Replace `TICKET_ID`, `TICKET_SUMMARY`, `BLOCKS`, `TEST_CASES`, `MOCKS`, and `AC_COVERAGE` with values for the current ticket.

All content must be in English.

```python
import xlsxwriter

TICKET_ID      = "PROJ-XXXX"
TICKET_SUMMARY = "Ticket Title"

BLOCKS = {
    # "Block Name": ("header_hex", "row_tint_hex")
}

# Each tuple: (block, tc_id, title, gherkin_desc, notes, automatizable)
# gherkin_desc: use \n between Given / When / Then lines
# automatizable: "Yes", "No", or "-"
TEST_CASES = []

# Key Variables: first line = "Used by: TC-XX, TC-YY"
#                second line = actual variable descriptions
MOCKS = [
    # ("Mock Name", "Used by: TC-04, TC-05\nkeyVar: value, keyVar2: value"),
]

# (ac_id, description, covered_by_tcs, is_covered)
AC_COVERAGE = []

TOTAL_TCS     = len(TEST_CASES)
OUT_PATH      = f"/tmp/{TICKET_ID}_TestCases.xlsx"
checkbox_rows = []   # populated below — 1-indexed rows for TC and Mock col-A cells

wb  = xlsxwriter.Workbook(OUT_PATH, {"strings_to_urls": False})
ws  = wb.add_worksheet("Test Cases")
ws2 = wb.add_worksheet("Block Summary")

def F(bold=False, color="111827", bg="FFFFFF", size=10,
      align="left", wrap=True, valign="vcenter"):
    return wb.add_format({
        "bold": bold, "font_color": color, "bg_color": bg,
        "font_size": size, "align": align, "valign": valign,
        "text_wrap": wrap, "font_name": "Arial",
        "border": 1, "border_color": "D1D5DB",
    })

# ── Shared formats ──────────────────────────────────────────────
fmt_title    = F(bold=True, color="FFFFFF", bg="1A56A0", size=14, align="center")
fmt_hdr      = F(bold=True, color="FFFFFF", bg="1F2937", size=11, align="center")
fmt_subhdr   = F(bold=True, color="FFFFFF", bg="4B5563", size=10, align="center")
fmt_ac_hdr   = F(bold=True, color="FFFFFF", bg="374151", size=10, align="center")
fmt_mock_sep = F(bold=True, color="FFFFFF", bg="374151", size=11, align="left")
fmt_ac_sep   = F(bold=True, color="FFFFFF", bg="1F2937", size=11, align="left")

# Mock row formats
fmt_mock_bool = F(bg="F9FAFB", align="center")
fmt_mock_name = F(bold=True, color="1F2937", bg="F9FAFB")
fmt_mock_vars = F(color="374151", bg="F9FAFB")   # wrap=True → auto-expand height

# AC row formats (covered / not covered)
fmt_ac_id_cov   = F(bold=True, color="166534", bg="D1FAE5", size=10, align="center")
fmt_ac_id_not   = F(bold=True, color="7C2D12", bg="FFEDD5", size=10, align="center")
fmt_ac_desc_cov = F(bg="D1FAE5")
fmt_ac_desc_not = F(bg="FFEDD5")
fmt_ac_tcs_cov  = F(color="374151", bg="D1FAE5")
fmt_ac_tcs_not  = F(color="374151", bg="FFEDD5")
fmt_ac_stat_cov = F(bold=True, color="166534", bg="D1FAE5", size=10, align="center")
fmt_ac_stat_not = F(bold=True, color="7C2D12", bg="FFEDD5", size=10, align="center")

# ── Per-block formats (pre-created outside loops) ───────────────
block_fmts = {}
for blk, (hx, rx) in BLOCKS.items():
    block_fmts[blk] = {
        "sep":    F(bold=True, color="FFFFFF", bg=hx, size=11, align="left"),
        "bool":   F(bg=rx, align="center"),
        "id":     F(bold=True, color=hx, bg=rx, size=10, align="center"),
        "cell":   F(bg=rx),
        "title":  F(bold=True, bg=rx),
        "note":   F(bg=rx, color="6B7280"),
        "status": F(bg=rx, align="center", color="374151"),
    }

# ── Column widths ───────────────────────────────────────────────
for i, w in enumerate([12, 9, 36, 22, 58, 32, 16, 18]):
    ws.set_column(i, i, w)

# ── Row 0: sheet title ──────────────────────────────────────────
ws.merge_range(0, 0, 0, 7,
    f"{TICKET_ID} — Test Cases: {TICKET_SUMMARY}  ·  {TOTAL_TCS} scenarios",
    fmt_title)
ws.set_row(0, 38)

# ── Row 1: column headers ───────────────────────────────────────
HEADERS = ["Passed ✓", "ID", "Block", "Scenario Title",
           "Description (Given / When / Then)", "Notes / Mock", "Status", "Automatable?"]
for ci, h in enumerate(HEADERS):
    ws.write(1, ci, h, fmt_hdr)
ws.set_row(1, 28)

# ── Data validations — TC rows only ────────────────────────────
tc_last_0i = 2 + len(BLOCKS) + len(TEST_CASES) - 1
ws.data_validation(2, 6, tc_last_0i, 6, {
    "validate": "list",
    "source": ["Not tested", "Passed", "Failed", "Blocked", "N/A"],
    "show_error": False,
})
ws.data_validation(2, 7, tc_last_0i, 7, {
    "validate": "list",
    "source": ["Yes", "No", "-"],
    "show_error": False,
})

# ── Test case rows ──────────────────────────────────────────────
current_block = None
row = 2  # 0-indexed

for (block, tc_id, title, desc, notes, auto) in TEST_CASES:
    fmts = block_fmts[block]

    if block != current_block:
        current_block = block
        ws.merge_range(row, 0, row, 7, f"  {block}", fmts["sep"])
        ws.set_row(row, 22)
        row += 1

    checkbox_rows.append(row + 1)    # 1-indexed; used by Step 6.3 for Sheets API checkboxes
    ws.write(row, 0, False,          fmts["bool"])
    ws.write(row, 1, tc_id,          fmts["id"])
    ws.write(row, 2, block,          fmts["cell"])
    ws.write(row, 3, title,          fmts["title"])
    ws.write(row, 4, desc,           fmts["cell"])
    ws.write(row, 5, notes or "—",   fmts["note"])
    ws.write(row, 6, "Not tested",   fmts["status"])
    ws.write(row, 7, auto or "-",    fmts["status"])
    # DO NOT call set_row() — Google Sheets auto-expands wrapped rows
    row += 1

# Empty separator between last TC and Mock block — no writes, no set_row()
row += 1

# ── Mock checklist block ────────────────────────────────────────
mock_block_num = len(BLOCKS) + 1
ws.merge_range(row, 0, row, 7,
    f"  BLOCK {mock_block_num} — Mock Checklist  (confirm with dev if all are implemented)",
    fmt_mock_sep)
ws.set_row(row, 24)
row += 1

# Mock sub-header: A | B:C | D:H
ws.write(row, 0, "Confirmed?", fmt_subhdr)
ws.merge_range(row, 1, row, 2, "Mock", fmt_subhdr)
ws.merge_range(row, 3, row, 7, "Key Variables", fmt_subhdr)
ws.set_row(row, 22)
row += 1

for (mock_name, mock_vars) in MOCKS:
    checkbox_rows.append(row + 1)    # 1-indexed
    ws.write(row, 0, False, fmt_mock_bool)
    ws.merge_range(row, 1, row, 2, mock_name, fmt_mock_name)
    ws.merge_range(row, 3, row, 7, mock_vars, fmt_mock_vars)
    # DO NOT call set_row() — auto-expands when Key Variables wraps
    row += 1

# ── AC Coverage block ───────────────────────────────────────────
if AC_COVERAGE:
    row += 1
    ws.merge_range(row, 0, row, 7, "  AC Coverage", fmt_ac_sep)
    ws.set_row(row, 22)
    row += 1

    # Sub-header: A | B:C | D | E:H
    ws.write(row, 0, "AC", fmt_ac_hdr)
    ws.merge_range(row, 1, row, 2, "Description", fmt_ac_hdr)
    ws.write(row, 3, "Covered by", fmt_ac_hdr)
    ws.merge_range(row, 4, row, 7, "Status", fmt_ac_hdr)   # E:H — no empty G/H cells
    ws.set_row(row, 20)
    row += 1

    for (ac_id, ac_desc, tcs, is_covered) in AC_COVERAGE:
        f_id   = fmt_ac_id_cov   if is_covered else fmt_ac_id_not
        f_desc = fmt_ac_desc_cov if is_covered else fmt_ac_desc_not
        f_tcs  = fmt_ac_tcs_cov  if is_covered else fmt_ac_tcs_not
        f_stat = fmt_ac_stat_cov if is_covered else fmt_ac_stat_not
        status_val = "Covered" if is_covered else "Not covered"

        ws.write(row, 0, ac_id,        f_id)
        ws.merge_range(row, 1, row, 2, ac_desc,       f_desc)
        ws.write(row, 3, tcs or "—",   f_tcs)
        ws.merge_range(row, 4, row, 7, status_val,    f_stat)   # E:H merged
        # DO NOT call set_row() — auto-expand
        row += 1

ws.freeze_panes(2, 0)

# ── Block Summary sheet ─────────────────────────────────────────
ws2.set_column(0, 0, 50)
ws2.set_column(1, 1, 12)
ws2.set_column(2, 2, 12)

ws2.merge_range(0, 0, 0, 2,
    f"Block Summary — {TICKET_ID}",
    F(bold=True, color="FFFFFF", bg="1A56A0", size=13, align="center"))
ws2.set_row(0, 30)

for ci, h in enumerate(["Block", "TCs", "Color"]):
    ws2.write(1, ci, h, F(bold=True, color="FFFFFF", bg="1F2937", size=10, align="center"))

counts = {}
for (blk, *_) in TEST_CASES:
    counts[blk] = counts.get(blk, 0) + 1

for i, (blk, cnt) in enumerate(counts.items(), 2):
    hx, rx = BLOCKS.get(blk, ("555555", "F9FAFB"))
    ws2.write(i, 0, blk, F(bg=rx, color="111827"))
    ws2.write(i, 1, cnt, F(bold=True, color=hx, bg=rx, size=11, align="center"))
    ws2.write(i, 2, "",  F(bg=hx))   # color swatch

tr = len(counts) + 2
ws2.write(tr, 0, "TOTAL",     F(bold=True, color="FFFFFF", bg="1F2937", align="center"))
ws2.write(tr, 1, TOTAL_TCS,   F(bold=True, color="FFFFFF", bg="1F2937", align="center"))

wb.close()
print(f"File ready: {OUT_PATH}")
print(f"Checkbox rows (1-indexed): {checkbox_rows}")
# Pass checkbox_rows to Step 6.3 for Sheets API batchUpdate
```

## Important Notes

- Install xlsxwriter if needed: `pip install xlsxwriter --break-system-packages`
- Output path: `/tmp/{TICKET_ID}_TestCases.xlsx`
- **Column A**: write Python `False` directly — xlsxwriter emits a proper XLSX boolean cell. No DataValidation needed. User selects column A → Insert > Checkbox in Google Sheets.
- **Row heights**: never call `ws.set_row()` on TC rows, Mock rows, or AC Coverage rows — Google Sheets auto-expands rows whose cells have `text_wrap=True` and no locked height.
- **Data validation scope**: `tc_last_0i = 2 + len(BLOCKS) + len(TEST_CASES) - 1` (0-indexed). The empty separator row and all rows below are excluded.
- **Mock Key Variables**: first line = `"Used by: TC-XX, TC-YY"`, second line = key variable list, separated by `\n`.
- **AC Coverage layout**: A=AC ID, B:C=Description, D=Covered by, E:H=Status. G and H are always part of the Status merge — never left as empty styled cells.
- **All content must be in English** — TC data, mock names, notes, AC descriptions, and all spreadsheet labels.
