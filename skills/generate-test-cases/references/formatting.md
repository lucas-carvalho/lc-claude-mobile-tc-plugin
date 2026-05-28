# Spreadsheet Formatting Reference — openpyxl

## General File Structure

- **Main sheet**: `"Test Cases"` — contains all TCs
- **Secondary sheet**: `"Block Summary"` — TC count per block with color legend
- Freeze panes at `A3` (row 1 = title, row 2 = column headers)

## Columns (A–H)

| Col | Name | Width | Alignment | Notes |
|-----|------|-------|-----------|-------|
| A | Passed ✓ | 12 | Center | Boolean `False` value — no data validation; user converts to checkbox in GSheets |
| B | ID | 9 | Center | |
| C | Block | 36 | Left | |
| D | Scenario Title | 22 | Left | |
| E | Description (Given / When / Then) | 58 | Left | Gherkin format, wrap enabled |
| F | Notes / Mock | 32 | Left | Mock data, AC ref, gaps |
| G | Status | 16 | Center | Dropdown: Not tested / Passed / Failed / Blocked / N/A |
| H | Automatable? | 18 | Center | Dropdown: Yes / No / - (wider to fit label on one line) |

## Special Rows

- **Row 1**: Sheet title — merged **A1:H1**, blue background `1A56A0`, white bold 14pt font, height 38
- **Row 2**: Column headers — dark background `1F2937`, white bold 11pt font, height 28
- **TC rows**: NO custom height — omit `ws.row_dimensions[row].height` so Google Sheets auto-expands
- **Block separator rows**: merged **A:H**, block color background, white bold 11pt font, height 22
- **Mock rows**: height 20 (fixed — content is always single-line)
- **AC Coverage rows**: NO custom height — same reason as TC rows

## Google Sheets Compatibility Rules

1. **Column A checkbox**: Do NOT use `DataValidation(type="list", formula1='"TRUE,FALSE"')` — Google Sheets renders this as a text dropdown. Instead, write Python boolean `False` as the cell value with no data validation. In Google Sheets the user can then select the column and use **Insert > Checkbox** to convert them to native interactive checkboxes in one click.

2. **Row heights**: Do NOT set a fixed `ws.row_dimensions[row].height` on TC rows or AC Coverage rows. A fixed height in xlsx is respected by Google Sheets as a locked height — wrap text will NOT auto-expand the row. Without a fixed height, Google Sheets calculates the row height from the wrapped content automatically.

3. **Data validation scope**: Apply `dv_status` (G) and `dv_auto` (H) dropdowns **only to TC rows** — not to mock checklist rows or AC Coverage rows. Calculate `tc_last_row = 2 + len(BLOCKS) + len(TEST_CASES)` and use that as the upper bound for both validations.

## Data Validations

```python
from openpyxl.worksheet.datavalidation import DataValidation

# Column A: NO data validation — use boolean False value (see Google Sheets compat above)

# Scope validations to TC rows only
tc_last_row = 2 + len(BLOCKS) + len(TEST_CASES)

# Column G (Status dropdown)
dv_status = DataValidation(type="list",
    formula1='"Not tested,Passed,Failed,Blocked,N/A"', allow_blank=True)
dv_status.sqref = f"G3:G{tc_last_row}"
ws.add_data_validation(dv_status)

# Column H (Automatable — Yes/No/- in English)
dv_auto = DataValidation(type="list", formula1='"Yes,No,-"', allow_blank=True)
dv_auto.sqref = f"H3:H{tc_last_row}"
ws.add_data_validation(dv_auto)
```

## Block Color Palette

```python
# Format: "Block Name": ("header_hex", "row_tint_hex")
COLOR_PALETTE = {
    "access":        ("1A56A0", "DDEEFF"),   # Access / Navigation (blue)
    "structure":     ("92400E", "FEF3C7"),   # Structure / Order (amber)
    "ni_v1":         ("7B2D8B", "F0E0FF"),   # Not Started — variant 1 (purple)
    "ni_v2":         ("166534", "D1FAE5"),   # Not Started — variant 2 (dark green)
    "ni_v3":         ("065666", "CCFBF1"),   # Not Started — variant 3 (teal)
    "ip_v1":         ("3D1A78", "EDE9FE"),   # In Progress — variant 1 (violet)
    "ip_v2":         ("1E3A5F", "DBEAFE"),   # In Progress — variant 2 (medium blue)
    "ip_v3":         ("14532D", "DCFCE7"),   # In Progress — variant 3 (mint green)
    "completed":     ("7C2D12", "FFEDD5"),   # Completed (red/orange)
    "types":         ("374151", "F3F4F6"),   # Card Types / Labels (slate)
    "edge":          ("1E3A8A", "EFF6FF"),   # Edge Cases (royal blue)
    "segment":       ("065666", "CCFBF1"),   # Segment events (teal)
    "firebase":      ("92400E", "FEF3C7"),   # Firebase / Auth (burnt orange)
    "revenuecat":    ("4C1D95", "EDE9FE"),   # RevenueCat / Paywall (deep purple)
    "nonfunc":       ("1E4D2B", "D1FAE5"),   # Non-functional / Mobile quality (forest green)
    "mock":          ("374151", "F9FAFB"),   # Mock Checklist (neutral gray)
    "coverage":      ("1F2937", "F3F4F6"),   # AC Coverage Summary (dark)
}
```

## Complete Python Script Template

Replace `TICKET_ID`, `TICKET_SUMMARY`, `BLOCKS`, `TEST_CASES`, `MOCKS`, and `AC_COVERAGE` with values for the current ticket.

All content must be in English — scenario titles, Gherkin descriptions, notes, mock variable names, and labels.

```python
import openpyxl
from openpyxl.styles import PatternFill, Font, Alignment, Border, Side
from openpyxl.utils import get_column_letter
from openpyxl.worksheet.datavalidation import DataValidation

wb = openpyxl.Workbook()
ws = wb.active
ws.title = "Test Cases"

TICKET_ID = "PROJ-XXXX"
TICKET_SUMMARY = "Ticket Title"

BLOCKS = {
    # "Block Name": ("header_hex", "row_tint_hex")
}

# Each tuple: (block_name, tc_id, title, gherkin_description, notes, automatizable)
# gherkin_description: use \n between Given/When/Then lines
# automatizable: "Yes", "No", or "-"
TEST_CASES = [
    # ("Block Name", "TC-01", "Short title", "Given ...\nWhen ...\nThen ...", "Ref: AC-1 / notes", "Yes"),
]

MOCKS = [
    # ("MockName", "Key variables in English"),
]

# List of tuples: (ac_id, description, covered_by_tcs, is_covered)
AC_COVERAGE = [
    # ("AC-1", "AC description in English", "TC-01, TC-02", True),
    # ("AC-2", "AC description in English", "", False),  # uncovered
]

TOTAL_TCS = len(TEST_CASES)

# ── Style helpers ──────────────────────────────────────
def fill(h):      return PatternFill("solid", fgColor=h)
def bold_white(s=11): return Font(bold=True, color="FFFFFF", size=s, name="Arial")
def regular(color="111827", s=10): return Font(color=color, size=s, name="Arial")
thin   = Side(border_style="thin",   color="D1D5DB")
medium = Side(border_style="medium", color="9CA3AF")
def bdr(l=thin,r=thin,t=thin,b=thin): return Border(left=l,right=r,top=t,bottom=b)
def center(wrap=True): return Alignment(horizontal="center", vertical="center", wrap_text=wrap)
def left(wrap=True):   return Alignment(horizontal="left",   vertical="center", wrap_text=wrap)

# Columns A–H widths
# A=12 for "Passed ✓"; H=18 so "Automatable?" fits on one line
COL_WIDTHS = [12, 9, 36, 22, 58, 32, 16, 18]
for cl, w in zip([get_column_letter(i) for i in range(1, 9)], COL_WIDTHS):
    ws.column_dimensions[cl].width = w

# ── Sheet title (row 1, merged A1:H1) ─────────────────
ws.merge_cells("A1:H1")
c = ws["A1"]
c.value = f"{TICKET_ID} — Test Cases: {TICKET_SUMMARY}  ·  {TOTAL_TCS} scenarios"
c.fill  = fill("1A56A0")
c.font  = Font(bold=True, color="FFFFFF", size=14, name="Arial")
c.alignment = center()
ws.row_dimensions[1].height = 38

# ── Column headers (row 2) ────────────────────────────
# "Passed ✓" — check mark after label; H widened to prevent "Automatable?" wrapping
HEADERS = ["Passed ✓", "ID", "Block", "Scenario Title",
           "Description (Given / When / Then)", "Notes / Mock", "Status", "Automatable?"]
for ci, h in enumerate(HEADERS, 1):
    c = ws.cell(row=2, column=ci, value=h)
    c.fill = fill("1F2937"); c.font = bold_white(11)
    c.alignment = center(); c.border = bdr()
ws.row_dimensions[2].height = 28

# ── Data validations — TC rows only ──────────────────
# Scoped to TC rows only; Mock and AC Coverage sections excluded intentionally.
tc_last_row = 2 + len(BLOCKS) + len(TEST_CASES)
dv_status = DataValidation(type="list",
    formula1='"Not tested,Passed,Failed,Blocked,N/A"', allow_blank=True)
dv_status.sqref = f"G3:G{tc_last_row}"
ws.add_data_validation(dv_status)
dv_auto = DataValidation(type="list", formula1='"Yes,No,-"', allow_blank=True)
dv_auto.sqref = f"H3:H{tc_last_row}"
ws.add_data_validation(dv_auto)

# ── Test case rows ────────────────────────────────────
current_block = None
row = 3

for (block, tc_id, title, desc, notes, auto) in TEST_CASES:
    hx, rx = BLOCKS.get(block, ("374151", "F9FAFB"))

    # Block separator row (merged A:H)
    if block != current_block:
        current_block = block
        ws.merge_cells(f"A{row}:H{row}")
        sep = ws.cell(row=row, column=1, value=f"  {block}")
        sep.fill      = fill(hx)
        sep.font      = Font(bold=True, color="FFFFFF", size=11, name="Arial")
        sep.alignment = left()
        sep.border    = bdr(t=Side(border_style="medium", color="FFFFFF"))
        ws.row_dimensions[row].height = 22
        row += 1

    def tc_cell(col, val, **kw):
        c = ws.cell(row=row, column=col, value=val)
        c.fill = fill(rx); c.border = bdr()
        c.alignment = kw.get("align", left())
        c.font = kw.get("font", regular())
        return c

    # Column A: boolean False — user selects column in GSheets > Insert > Checkbox
    tc_cell(1, False, align=center())
    tc_cell(2, tc_id, font=Font(bold=True, color=hx, size=10, name="Arial"), align=center())
    tc_cell(3, block, font=regular("374151"))
    tc_cell(4, title, font=Font(bold=True, color="111827", size=10, name="Arial"))
    tc_cell(5, desc)
    tc_cell(6, notes if notes else "—", font=regular("6B7280"))
    tc_cell(7, "Not tested", align=center(), font=regular("374151"))
    tc_cell(8, auto if auto else "-", align=center(), font=regular("374151"))
    # NOTE: Do NOT set ws.row_dimensions[row].height — let Google Sheets auto-expand.
    row += 1

# ── Mock checklist block ──────────────────────────────
row += 1
mock_block_num = len(BLOCKS) + 1
ws.merge_cells(f"A{row}:H{row}")
b_mock = ws.cell(row=row, column=1,
    value=f"  BLOCK {mock_block_num} — Mock Checklist  (confirm with dev if all are implemented)")
b_mock.fill = fill("374151"); b_mock.font = bold_white(11)
b_mock.alignment = left(); b_mock.border = bdr()
ws.row_dimensions[row].height = 24
row += 1

# Mock sub-header: B:C merged for "Mock", D:H merged for "Key Variables"
ws.merge_cells(f"B{row}:C{row}")
ws.merge_cells(f"D{row}:H{row}")
for (col, val) in [(1, "Confirmed?"), (2, "Mock"), (4, "Key Variables")]:
    c = ws.cell(row=row, column=col, value=val)
    c.fill = fill("4B5563"); c.font = bold_white(10)
    c.alignment = center(); c.border = bdr()
for col in [3, 5, 6, 7, 8]:
    c = ws.cell(row=row, column=col)
    c.fill = fill("4B5563"); c.border = bdr()
ws.row_dimensions[row].height = 22
row += 1

for (mock_name, mock_vars) in MOCKS:
    # Column A: boolean False (same as TC rows)
    c_a = ws.cell(row=row, column=1, value=False)
    c_a.fill = fill("F9FAFB"); c_a.alignment = center(); c_a.border = bdr()
    ws.merge_cells(f"B{row}:C{row}")
    mn = ws.cell(row=row, column=2, value=mock_name)
    mn.fill = fill("F9FAFB"); mn.font = Font(bold=True, color="1F2937", size=10, name="Arial")
    mn.alignment = left(); mn.border = bdr()
    ws.cell(row=row, column=3).border = bdr()
    ws.merge_cells(f"D{row}:H{row}")
    mv = ws.cell(row=row, column=4, value=mock_vars)
    mv.fill = fill("F9FAFB"); mv.font = regular("374151")
    mv.alignment = left(); mv.border = bdr()
    ws.row_dimensions[row].height = 20
    row += 1

# ── AC Coverage Summary block ─────────────────────────
if AC_COVERAGE:
    row += 1
    ws.merge_cells(f"A{row}:H{row}")
    b_ac = ws.cell(row=row, column=1,
        value="  AC Coverage (based on ticket description — no formal ACs defined)")
    #   OR: "  AC Coverage" if the ticket had formal ACs
    b_ac.fill = fill("1F2937"); b_ac.font = bold_white(11)
    b_ac.alignment = left(); b_ac.border = bdr()
    ws.row_dimensions[row].height = 22
    row += 1

    # AC sub-header: B:C merged for "Description", E:F merged for "Status"
    ws.merge_cells(f"B{row}:C{row}")
    ws.merge_cells(f"E{row}:F{row}")
    for (col, val) in [(1, "AC"), (2, "Description"), (4, "Covered by"), (5, "Status")]:
        c = ws.cell(row=row, column=col, value=val)
        c.fill = fill("374151"); c.font = bold_white(10)
        c.alignment = center(); c.border = bdr()
    for col in [3, 6, 7, 8]:
        c = ws.cell(row=row, column=col)
        c.fill = fill("374151"); c.border = bdr()
    ws.row_dimensions[row].height = 20
    row += 1

    for (ac_id, ac_desc, tcs, is_covered) in AC_COVERAGE:
        status_val   = "Covered" if is_covered else "Not covered"
        status_color = "166534" if is_covered else "7C2D12"
        rx           = "D1FAE5" if is_covered else "FFEDD5"

        c1 = ws.cell(row=row, column=1, value=ac_id)
        c1.fill=fill(rx); c1.font=Font(bold=True,color=status_color,size=10,name="Arial")
        c1.alignment=center(); c1.border=bdr()

        ws.merge_cells(f"B{row}:C{row}")
        c2 = ws.cell(row=row, column=2, value=ac_desc)
        c2.fill=fill(rx); c2.font=regular("111827"); c2.alignment=left(); c2.border=bdr()
        ws.cell(row=row, column=3).border = bdr()

        c4 = ws.cell(row=row, column=4, value=tcs if tcs else "—")
        c4.fill=fill(rx); c4.font=regular("374151"); c4.alignment=left(); c4.border=bdr()

        ws.merge_cells(f"E{row}:F{row}")
        c5 = ws.cell(row=row, column=5, value=status_val)
        c5.fill=fill(rx); c5.font=Font(bold=True,color=status_color,size=10,name="Arial")
        c5.alignment=center(); c5.border=bdr()
        ws.cell(row=row, column=6).fill=fill(rx); ws.cell(row=row, column=6).border=bdr()

        # G and H: filled but NO data validation (scoped to TC rows only)
        for col in [7, 8]:
            cx = ws.cell(row=row, column=col)
            cx.fill=fill(rx); cx.border=bdr()

        # NOTE: Do NOT set ws.row_dimensions[row].height — let Google Sheets auto-expand.
        row += 1

ws.freeze_panes = "A3"

# ── Summary sheet ─────────────────────────────────────
ws2 = wb.create_sheet("Block Summary")
ws2.column_dimensions["A"].width = 50
ws2.column_dimensions["B"].width = 12
ws2.column_dimensions["C"].width = 12

ws2.merge_cells("A1:C1")
t = ws2["A1"]
t.value = f"Block Summary — {TICKET_ID}"
t.fill = fill("1A56A0"); t.font = bold_white(13)
t.alignment = center()
ws2.row_dimensions[1].height = 30

for ci, h in enumerate(["Block", "TCs", "Color"], 1):
    c = ws2.cell(row=2, column=ci, value=h)
    c.fill = fill("1F2937"); c.font = bold_white(10)
    c.alignment = center(); c.border = bdr()

counts = {}
for (blk, *_) in TEST_CASES:
    counts[blk] = counts.get(blk, 0) + 1

for i, (blk, cnt) in enumerate(counts.items(), 3):
    hx, rx = BLOCKS.get(blk, ("555555","F9FAFB"))
    c1 = ws2.cell(row=i, column=1, value=blk)
    c1.fill=fill(rx); c1.font=regular("111827"); c1.border=bdr(); c1.alignment=left()
    c2 = ws2.cell(row=i, column=2, value=cnt)
    c2.fill=fill(rx); c2.font=Font(bold=True,color=hx,size=11,name="Arial")
    c2.border=bdr(); c2.alignment=center()
    c3 = ws2.cell(row=i, column=3)
    c3.fill=fill(hx); c3.border=bdr()

tr = len(counts)+3
ws2.cell(row=tr,column=1,value="TOTAL").fill=fill("1F2937")
ws2.cell(row=tr,column=1).font=bold_white(); ws2.cell(row=tr,column=1).border=bdr()
ws2.cell(row=tr,column=1).alignment=center()
ws2.cell(row=tr,column=2,value=TOTAL_TCS).fill=fill("1F2937")
ws2.cell(row=tr,column=2).font=bold_white(); ws2.cell(row=tr,column=2).border=bdr()
ws2.cell(row=tr,column=2).alignment=center()

OUT = f"/sessions/<session-name>/mnt/outputs/{TICKET_ID}_TestCases.xlsx"
wb.save(OUT)
print(f"Saved: {OUT}")
```

## Important Notes

- Install openpyxl if needed: `pip install openpyxl --break-system-packages`
- Output path: `/sessions/<current-session>/mnt/outputs/<TICKET_ID>_TestCases.xlsx` — determine the session name dynamically, do not hardcode it
- **Column A**: write Python boolean `False` — no DataValidation. In Google Sheets the user selects column A → Insert > Checkbox to get interactive checkboxes
- **TC and AC row heights**: do NOT set fixed heights — let Google Sheets auto-expand rows with wrapped content
- **Data validation scope**: use `tc_last_row = 2 + len(BLOCKS) + len(TEST_CASES)` as the upper bound — never extend validations into Mock or AC Coverage rows
- **Mock sub-header**: merge B:C for "Mock", merge D:H for "Key Variables"; write only to anchor cells (B, D) after merging
- **AC sub-header**: merge B:C for "Description", merge E:F for "Status"; write only to anchor cells (B, E) after merging
- Column H ("Automatable?") width is 18 — prevents the label from wrapping to two lines
- Column A ("Passed ✓") width is 12 — gives room for the label without wrapping
- **All content must be in English** — TC data, mock names, notes, AC descriptions, and all spreadsheet labels
