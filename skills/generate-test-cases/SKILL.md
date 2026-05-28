---
name: generate-test-cases
description: >
  This skill should be used when the user asks to "generate test cases", "create test cases",
  "generate a QA spreadsheet", "test this ticket", "create tests for a ticket", "analyze a Jira ticket",
  or provides a Jira ticket ID or URL (e.g. "PROJ-1234", "https://yourorg.atlassian.net/browse/PROJ-1234").
  Also triggers on "test case spreadsheet", "QA sheet", or any request to generate QA scenarios
  for a mobile app ticket from a configured Jira board.
metadata:
  version: "0.5.2"
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

The settings file is stored in the **persistent** Cowork directory so it survives across sessions. Locate it dynamically — do NOT hardcode a session name:

```bash
SETTINGS=$(find /sessions -maxdepth 4 -name "lc-mobile-qa-settings.json" -path "*/mnt/.claude/*" 2>/dev/null | head -1)
[ -f "$SETTINGS" ] && cat "$SETTINGS"
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
- Write the settings file to the **persistent** Cowork directory via Bash (find it dynamically):

```bash
MNT_CLAUDE=$(find /sessions -maxdepth 3 -path "*/mnt/.claude" -type d 2>/dev/null | head -1)
cat > "$MNT_CLAUDE/lc-mobile-qa-settings.json" << 'EOF'
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
```

Confirm with the user: "Configuration saved. You won't need to do this again." Then proceed.

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

## Step 6: Upload to Google Drive

After the file is generated, upload it to the configured Drive folder using the Drive MCP:
- Use the `folderId` from the settings loaded in Step 0
- Tool: `mcp__0948f3c6-7a85-418d-ab1c-3fdf6d0cd2b2__create_file` (or equivalent Drive MCP create tool)

If the Drive upload fails or the MCP tool is unavailable, inform the user and present the local `.xlsx` file for manual upload.

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
