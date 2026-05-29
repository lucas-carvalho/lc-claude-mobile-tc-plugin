---
name: generate-test-cases
description: >
  This skill should be used when the user asks to "generate test cases", "create test cases",
  "generate a QA spreadsheet", "test this ticket", "create tests for a ticket", "analyze a Jira ticket",
  or provides a Jira ticket ID or URL (e.g. "PROJ-1234", "https://yourorg.atlassian.net/browse/PROJ-1234").
  Also triggers on "test case spreadsheet", "QA sheet", or any request to generate QA scenarios
  for a mobile app ticket from a configured Jira board.
metadata:
  version: "0.6.1"
  author: "Lucas Carvalho"
---

# Skill: Generate Test Cases — Mobile App QA (Shift Left)

Generate a formatted, color-coded test case spreadsheet for a Jira ticket and save it to Google Drive.

This skill follows a **Shift Left** quality approach: quality concerns are raised as early as possible, test cases are grounded in Acceptance Criteria, and coverage gaps are surfaced before testing begins.

> **Language rule**: The entire spreadsheet — structural elements (column headers, block labels, status values, dropdown options) AND TC content (scenario titles, Gherkin descriptions, notes, mock names) — must always be written in **English**. No Portuguese or other languages should appear anywhere in the spreadsheet output.

---

## Step 0: Load Configuration

Record the start time immediately so elapsed duration can be reported in Step 7:

```bash
START_TIME=$(date +%s)
echo "⏱ Start: $(date '+%Y-%m-%d %H:%M:%S')"
```

Before anything else, check if the plugin has been configured for this user.

The settings file should live in a **persistent, writable** directory so it survives across sessions. Some Cowork session layouts mount `.claude/` as read-only — the discovery below tries multiple candidates and picks the first writable one. Do NOT hardcode a session name.

```bash
# Reading: try all known locations, take the first that has the file
SETTINGS=""
for candidate in \
    $(find /sessions -maxdepth 4 -name "lc-mobile-qa-settings.json" -path "*/mnt/.claude/*" 2>/dev/null) \
    $(find /sessions -maxdepth 4 -name "lc-mobile-qa-settings.json" -path "*/mnt/outputs/*" 2>/dev/null) \
    "$HOME/.claude/lc-mobile-qa-settings.json" \
    "$HOME/.lc-mobile-qa-settings.json"; do
  if [ -f "$candidate" ] && [ -s "$candidate" ]; then
    SETTINGS="$candidate"
    break
  fi
done
[ -n "$SETTINGS" ] && cat "$SETTINGS"
```

- If the command **returns valid JSON**, extract `jira.cloudId`, `jira.site`, `jira.projectKey`, and `drive.folderId`. Proceed to Step 1.
- If the command **returns nothing or an error**, run the **First-Run Setup** below before proceeding.

### First-Run Setup

Tell the user:

> "To get started, I need to configure two things: your Jira board and your Google Drive output folder. This only happens once."

Ask the following questions using AskUserQuestion:

1. **Jira board URL** — Ask for the full URL of their Jira board (e.g. `https://yourorg.atlassian.net/jira/software/c/projects/MYPROJECT/boards/123`). Parse the `projectKey` and `cloudId` from it, or use the Atlassian MCP `getAccessibleAtlassianResources` tool to look up the `cloudId` for their site.

2. **Google Drive folder** — Ask for the URL or folder ID of the Google Drive folder where spreadsheets should be saved. Parse the folder ID from a Drive URL if provided (the segment after `/folders/`).

Once the user provides both:
- Derive `jira.site` from the board URL (e.g. `yourorg.atlassian.net`)
- Resolve `jira.cloudId` via Atlassian MCP if not determinable from the URL alone
- Pick the **first writable** persistent directory and write the settings there. Fail loud, never silent:

```bash
# Writing: pick the first writable persistent candidate
SETTINGS_DIR=""
for candidate in \
    $(find /sessions -maxdepth 3 -path "*/mnt/.claude" -type d 2>/dev/null) \
    "$HOME/.claude"; do
  if [ -d "$candidate" ] && [ -w "$candidate" ]; then
    SETTINGS_DIR="$candidate"; break
  fi
done

# Fallback: create $HOME/.claude if nothing else worked
if [ -z "$SETTINGS_DIR" ]; then
  mkdir -p "$HOME/.claude" 2>/dev/null && [ -w "$HOME/.claude" ] && SETTINGS_DIR="$HOME/.claude"
fi

# Last resort: ephemeral outputs/ — warn the user that re-setup will be needed next session
if [ -z "$SETTINGS_DIR" ]; then
  SETTINGS_DIR=$(find /sessions -maxdepth 3 -path "*/mnt/outputs" -type d 2>/dev/null | head -1)
  echo "⚠ No writable persistent directory found. Caching settings to $SETTINGS_DIR (ephemeral)."
fi

[ -z "$SETTINGS_DIR" ] && { echo "ERROR: no writable location for settings"; exit 1; }

cat > "$SETTINGS_DIR/lc-mobile-qa-settings.json" << 'EOF'
{
  "jira": {
    "cloudId": "<resolved-cloud-id>",
    "site": "<atlassian-site>",
    "projectKey": "<PROJECT_KEY>",
    "boardUrl": "<full-board-url>"
  },
  "drive": {
    "folderId": "<folder-id>",
    "folderUrl": "<full-drive-url-if-provided>"
  }
}
EOF
echo "✓ Settings written to: $SETTINGS_DIR/lc-mobile-qa-settings.json"
```

Confirm with the user: "Configuration saved to `$SETTINGS_DIR`. You won't need to do this again." (Include the actual path so the user can spot if it ended up in an ephemeral location.) Then proceed.

---

## Step 1: Identify the Ticket

Extract the ticket ID from the user's message (e.g. `PROJ-1234`).
If the user provides a full URL, parse the ticket key from the path.
If no ticket ID is given, ask: "Which ticket ID or URL should I generate test cases for?"

---

## Step 2: Fetch Ticket from Jira

Use the Atlassian MCP `getJiraIssue` tool with:
- `cloudId` from config
- The extracted ticket key

Read and extract the following:
- **Summary, description, issue type, labels, status, components**
- **Acceptance Criteria** — look for a dedicated AC field, a "Definition of Done" section, or a structured list within the description. Extract each AC as a numbered list for use in Step 2.5 and Step 4.
- **Linked issues** (sub-tasks, parent epic, related bugs)
- **Attachments** — note any linked Figma designs or PR references

---

## Step 2.5: Pre-Quality Check (Shift Left Signal)

Before generating any test cases, evaluate the ticket's quality baseline. Flag any of the following issues and report them to the user:

| Condition | Warning to show |
|-----------|----------------|
| No Acceptance Criteria found | ⚠️ "No Acceptance Criteria found. TCs will be based on the ticket description — consider aligning ACs with the team before testing begins." |
| ACs exist but are vague or non-testable (e.g. "should work correctly") | ⚠️ "Some ACs appear ambiguous or untestable. Recommended: refine them with the team before starting the test cycle." |
| No design/Figma link on a UI ticket | ℹ️ "No design link found. If a visual spec exists, add it to the ticket for better test coverage." |
| Ticket description is very short (< 3 sentences) | ℹ️ "The ticket description is very brief. Generated TCs may not cover all relevant scenarios — review with the team." |

After showing the flags, ask the user: "Do you want to continue anyway, or would you prefer to update the ticket first?"

If the user confirms continuation, proceed. If there are no ACs on a Functional or Integration ticket, note explicitly which TCs are based on description assumptions rather than explicit criteria.

---

## Step 3: Classify the Ticket Type

Determine the ticket type from its content. The 4 types are:

- **Functional** — A complete feature integrated with the API; focus on user interaction and screen states.
- **Integration** — Endpoint mapping/consumption: GraphQL queries/mutations, Segment analytics events (Click/CTA/Status/Page View), Firebase (auth), RevenueCat (subscription management), or other SDKs.
- **Component** — A visual React Native component converted to multi-platform native; focus on visual variants, states, and props.
- **Bug** — A mapped error or feature blocker; may belong to any of the above categories.

Read `references/ticket-types.md` for detailed guidance on what to test for each type.

---

## Step 4: Generate Test Cases

Read `references/methodology.md` before generating any test cases.

Key rules:
- **No fixed count** — the number of TCs depends entirely on the ticket complexity and type.
- **AC-oriented** — aim to cover every Acceptance Criterion with at least one TC. Flag any uncovered ACs at the end; do not block generation because of missing or incomplete ACs.
- **Gherkin format** — write every TC description as `Given / When / Then` (see methodology.md for the full spec). This is the default format, not optional.
- **Functional/scenario perspective** — group TCs by screen state or user scenario, not by individual variable or unit.
- **Block naming** — use plain English names without emoji prefixes; color coding is applied through spreadsheet formatting only (see methodology.md Color Conventions).
- Each TC must have: ID (`TC-01`, `TC-02`...), short scenario title, Gherkin description, notes (mock data, preconditions, known gaps), and an automatable classification (`Yes`, `No`, or `-`).
- Include a **non-functional quality block** when relevant (see methodology.md for mobile-layer scope).
- Always end with a mock checklist block (`BLOCK N — Mock Checklist`) and an AC coverage summary.
- **All TC content must be in English** — scenario titles, Gherkin descriptions, notes, and mock names.

---

## Step 5: Generate the Spreadsheet

Read `references/formatting.md` for exact openpyxl code patterns.

1. Write a Python script that encodes the generated TCs using the template in `references/formatting.md`.
2. Install openpyxl if needed: `pip install openpyxl --break-system-packages`
3. Run the script via Bash to produce the `.xlsx` file.
4. Save the output to: `/sessions/<session-name>/mnt/outputs/<TICKET_ID>_TestCases.xlsx`
   - Determine the current session path dynamically — do not hardcode a session name.
5. Present the file to the user with `mcp__cowork__present_files`.

**Naming**: `<TICKET_ID>_TestCases.xlsx` — e.g. `PROJ-1234_TestCases.xlsx`.

---

## Step 6: Upload to Google Drive — Native Google Sheet in the Configured Folder

**Goal:** end with exactly one file in Drive — a native Google Sheet — inside the configured folder. Nothing in the Drive root, no leftover `.xlsx` in the folder.

### What this MCP can and can't do

The Drive MCP's `create_file` does NOT auto-convert XLSX to Google Sheets — per its docs, only `text/csv` and `text/plain` are auto-converted. The `disableConversionToGoogleType` flag has no effect for XLSX. **Confirmed by behavior, not assumed.**

To get a native Google Sheet we therefore drive the upload through the Drive **web UI** via Claude in Chrome. The web UI honors the user's account-level "Convert uploads to Google Docs editor format" setting (in `drive.google.com/drive/settings`), which auto-converts XLSX to GSheets server-side when the upload happens through the UI. The setting must be enabled — verify with the user once if uncertain.

### Path A — Primary: Chrome MCP via Drive Web UI

1. **Preflight.** Check Claude in Chrome is connected by calling `mcp__Claude_in_Chrome__list_connected_browsers`. If no browser is connected, jump to **Path B**.

2. **Set up a tab.** Call `tabs_context_mcp({ createIfEmpty: true })` to get the tab group. Call `tabs_create_mcp()` to claim a fresh tab; remember the returned `tabId`.

3. **Navigate to the configured folder:**

   ```
   navigate(tabId, url=`https://drive.google.com/drive/folders/${drive.folderId}`)
   ```

4. **Confirm the page is the right folder.** Call `get_page_text(tabId)`; the response should contain the folder name (read it from `drive.folderUrl` or scan the breadcrumb area). If Drive shows a login screen, stop and ask the user to sign in — do not attempt credential entry.

5. **Locate the drag-drop file input.** Call `find(tabId, query="hidden file upload input for drag and drop")`. Drive exposes an `<input type="file" multiple>` element that drag-drop targets — you want that element's `ref`. If `find` returns multiple matches, prefer one whose attributes suggest the upload zone for the current folder view.

6. **Upload via `file_upload`:**

   ```
   file_upload(tabId, ref=<input-ref>, paths=["/sessions/<session>/mnt/outputs/<TICKET_ID>_TestCases.xlsx"])
   ```

   This dispatches the file to Drive's upload handler. Because the Drive account has "Convert uploads" enabled, the result is a Google Sheet created **in the current folder**, not the root.

7. **Wait for the upload to settle.** Poll the Drive MCP every ~5s for up to 60s:

   ```
   search_files query:
     title contains '<TICKET_ID>_TestCases'
     and mimeType = 'application/vnd.google-apps.spreadsheet'
     and parentId = '<drive.folderId>'
   ```

   Stop polling as soon as a match returns. Capture its `id`, `name`, `mimeType`, `parents`, and `webViewLink`.

8. **Close the tab** once verified: `tabs_close_mcp(tabId)`.

### Path B — Fallback: Drive MCP upload (when Chrome MCP unavailable)

If Step 1 above showed no browser, or any step in Path A fails after one retry, fall back to:

1. Read the local `.xlsx` as base64 and call `mcp__0948f3c6-7a85-418d-ab1c-3fdf6d0cd2b2__create_file` with:
   - `title`: `"<TICKET_ID>_TestCases.xlsx"` (keep the suffix — it stays XLSX)
   - `base64Content`: base64 of the file
   - `contentMimeType`: `"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"`
   - `parentId`: `drive.folderId` (mandatory, never omit)
   - `disableConversionToGoogleType`: `true` (explicit — we know conversion isn't supported here)

2. Capture `id`, `mimeType`, `parents`, `webViewLink`. The result is an **XLSX** in the configured folder.

3. In Step 7's confirmation, include a one-line manual conversion instruction:

   > To convert to a native Google Sheet: open the file in Drive, then **File → Save as Google Sheets**. The converted copy lands in **My Drive** root — move it to the original folder via right-click → **Organize → Move**, then delete the `.xlsx`.

   Be explicit that the manual conversion lands in root by default — don't promise auto-placement.

---

## Step 6.5: Verify the Final State (Mandatory)

Independent of which path ran, before reporting success in Step 7, confirm via Drive MCP that exactly the right file exists in the right place.

1. Call `mcp__0948f3c6-7a85-418d-ab1c-3fdf6d0cd2b2__get_file_metadata` with the captured `fileId`.

2. Assert based on which path ran:

   **Path A (Chrome MCP):**
   - `mimeType == "application/vnd.google-apps.spreadsheet"`
   - `<drive.folderId>` appears in `parents`
   - `name == "<TICKET_ID>_TestCases"` or starts with that prefix (Drive may add a number suffix if a same-named file existed)

   **Path B (Drive MCP fallback):**
   - `mimeType == "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"` (XLSX preserved — expected)
   - `<drive.folderId>` appears in `parents`
   - `name == "<TICKET_ID>_TestCases.xlsx"`

3. (Path A only, optional) Call `read_file_content` on the `fileId` and check that `"Test Cases"` and `"Block Summary"` sheet names appear. This confirms the conversion preserved structure.

### If any assertion fails

- Do NOT report success. State which assertion failed and the actual value returned.
- Present the local `.xlsx` via `mcp__cowork__present_files` as a fallback the user can manually upload.
- If `parents` is wrong, show the actual `parents` array and the file URL so the user can move it.

---

## Step 6.6: Scan for Leftovers from Older Skill Versions

Versions before 0.6.1 sometimes left a duplicate Google Sheet at the Drive root (parentless `create_file` call). On every run, after a successful upload, scan for orphans for this ticket and surface them:

```
# Strays at the Drive root (most common older-version artifact)
search_files query:
  title contains '<TICKET_ID>_TestCases'
  and mimeType = 'application/vnd.google-apps.spreadsheet'
  and parentId = 'root'

# Stray XLSX duplicates in the configured folder
search_files query:
  title contains '<TICKET_ID>_TestCases'
  and mimeType contains 'spreadsheetml'
  and parentId = '<drive.folderId>'
```

Filter out the file you just uploaded (match against the captured `id`). List any other matches with their `webViewLink` URLs and ask: *"Found N older copy(ies) of this test case file. Want to clean them up? (You'll need to delete manually — see note below.)"*

> ⚠️ **Limitation:** the Drive MCP exposes no `delete_file` / `trash_file` tool. Cleanup is a manual user action via the Drive web UI. Do not promise auto-deletion.

The local `.xlsx` at `/sessions/<session>/mnt/outputs/` does not need cleanup — Cowork session storage is ephemeral.

---

## Step 7: Confirm

Briefly tell the user:
- Which ticket was analyzed and its classified type
- Pre-quality flags raised (if any)
- How many TCs were generated and the Blocks breakdown
- AC coverage summary (all ACs covered? any gaps?)
- Where the file was saved (Drive link if successful, or local path)

Also compute and report the elapsed time:

```bash
END_TIME=$(date +%s)
ELAPSED=$((END_TIME - START_TIME))
MINUTES=$((ELAPSED / 60))
SECONDS=$((ELAPSED % 60))
echo "⏱ Total time: ${MINUTES}m ${SECONDS}s"
```

Include the result inline in the confirmation message, e.g.: "Generated in **2m 14s**."

Do not repeat all TC content in chat — the spreadsheet is the deliverable.
